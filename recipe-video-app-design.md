# System Design for a Recipe Video Subscription App

## Overview

The planned service provides subscribers with on-demand recipe videos. Each recipe is composed of several small videos (2–4 GB total) and only paying users may access them. A React Native client will deliver the video experience, while a Ruby on Rails back-end will handle user management, subscription billing, administration, and video metadata. The application must be secure, scalable, cost-efficient and avoid vendor lock-in. Below is a detailed architecture and research on suitable cloud services.

## Front-End (React Native)

### Video playback
Use the react-native-video library. The library supports local files and remote sources and provides adaptive bitrate streaming (HLS and MPEG-DASH). Adaptive streaming dynamically adjusts video quality based on network conditions and device capabilities, which improves user experience. It also offers hooks to handle buffering and errors and can be integrated with custom controls. Best practices include:

- Prepare HLS/DASH streams instead of large MP4 downloads. HLS segments can adjust to bandwidth.
- Cache frequently accessed videos. The article notes that caching reduces repeated downloads and improves playback speed.
- Compress videos into optimized formats (e.g., H.264) to reduce file size without compromising quality.

### Authentication and subscription
The app should integrate a payment and subscription service. Stripe Billing supports recurring subscriptions, usage-based and one-time payments. It automates invoicing (pro-rations, dunning, reminders) and offers global payment methods and multi-currency support. Stripe's API allows custom workflows and analytics (MRR, churn). Pricing is 0.5% per recurring transaction (0.8% for advanced features) plus payment-processing fees.

### Analytics
Use a mobile analytics SDK to measure user engagement, screen flows and video events. A 2025 comparison identified Firebase Analytics (free, widely used on iOS/Android) as a top option. Mixpanel and Amplitude provide event-based and predictive analytics, while UXCam offers session replay. Integrate the chosen SDK in the React Native app and send important events (video start/completion, subscription status, etc.).

## Back-End (Ruby on Rails API)

### Authentication & Authorization

#### Built-in Rails authentication
Rails 8 includes an authentication generator. Running `rails generate authentication` adds models, controllers and views for login and password reset, and uses the bcrypt gem to securely hash passwords. After migration, visiting `/session/new` provides a sign-in form; however, you must implement your own sign-up flow.

#### Devise
For a fully featured authentication system, consider the Devise gem. It provides user sign-up, login, logout, password recovery and confirmation out of the box. Devise depends on bcrypt and adds salted, computationally expensive hashing to slow down brute-force attacks. You can add two-factor authentication with the devise-two-factor gem and integrate Role-Based Access Control (RBAC) using Pundit or policy objects.

#### Session and CSRF protection
Rails automatically includes CSRF tokens in non-GET requests to prevent cross-site request forgery (CSRF). Always render `<%= csrf_meta_tags %>` in layouts and ensure cookies are secure (HTTP-only, same-site). Force HTTPS by setting `config.force_ssl = true` and configure Nginx/Let's Encrypt for TLS. Rails 7.1+ supports Active Record Encryption so sensitive fields (e.g., email, phone) are encrypted at rest.

### Application Architecture

#### API-only Rails
Use Rails in API-only mode. The API will expose endpoints for user sign-up/sign-in, subscription management, recipe and video metadata, and analytics logging. Use JSON:API or GraphQL to structure responses.

#### Background jobs
Use Sidekiq or Active Job for tasks like video transcoding (via FFmpeg), generating thumbnails, sending emails, and generating pre-signed URLs.

#### Database
Use a relational database (PostgreSQL) to store user accounts, subscription status, recipe metadata, video metadata, and logs. Use a managed database service (e.g., DigitalOcean Managed PostgreSQL or AWS RDS) to avoid maintenance. Index columns used in queries (e.g., `user_id`, `recipe_id`) and normalise relations:

- **users**: id, email, password_digest/Devise fields, role, subscription_status, created_at, etc.
- **recipes**: id, title, description, total_duration, created_by
- **videos**: id, recipe_id (FK), storage_key (path to object), order_in_recipe, duration, file_size, created_at
- **sessions / log_entries**: record user sessions and key events (optional; see analytics section)

#### Caching
Use Rails caching to reduce database load and speed up responses:

- **Fragment caching** stores pieces of views that change infrequently. When different parts of a page have separate caching characteristics, fragment caching wraps the section in a cache block so subsequent requests use the cached version.
- **Russian doll caching** nests caches so that if an inner fragment (e.g., a video card) is updated, the outer list remains cached. This requires `touch: true` on associations so updating a child record also updates its parent and expires caches.
- **Low-level caching** uses `Rails.cache.fetch` to cache expensive computations (e.g., external API calls) with an expiry time. Use a cache store like Redis or Memcached.

### Security Best Practices

- **Use HTTPS and TLS** – Always run the application behind TLS. Set `config.force_ssl = true` and configure Nginx with Let's Encrypt certificates.
- **Encrypt sensitive data** – Use Active Record Encryption for columns storing personally identifiable information.
- **Role-Based Access Control** – Define roles (admin, contributor, subscriber) and enforce authorization with Pundit or policy objects.
- **Implement 2FA** – Integrate devise-two-factor to require a one-time password during login, adding an extra layer of security.
- **Secrets management** – Use Rails' encrypted credentials (`config/credentials.yml.enc`) to store API keys and secrets, keeping the master.key out of version control.
- **Web application firewall & DDoS protection** – Use Cloudflare or a similar CDN provider to mitigate DDoS and injection attacks. Cloudflare's CDN reduces latency by caching content near users and improves security by filtering malicious traffic.

## Video Storage & Delivery

### Storage Requirements
Each recipe totals 2–4 GB across multiple video files. For the MVP (≈50 recipes), the platform must store roughly 150–200 GB of data. The storage solution should scale as more recipes are added and avoid costly bandwidth fees.

### Object Storage Providers

| Provider | Storage Cost | Egress / API Fees | Notes |
|----------|-------------|-------------------|-------|
| **DigitalOcean Spaces** | Base subscription $5/month includes 250 GB storage and 1 TB outbound transfer; additional storage is $0.02/GB and bandwidth $0.01/GB | Up to 1 TB egress included; additional egress $0.01/GB | Simple fixed pricing; integrated CDN; 15 TB max object size; S3-compatible and includes a built-in CDN with edge caching |
| **Cloudflare R2** | $0.015/GB-month for standard storage; 10 GB free tier | Zero egress fees; 1M writes and 10M reads per month free; additional class A operations $4.50/M and class B operations $0.36/M | S3-compatible; integrates with Cloudflare's CDN; free egress ideal for frequent downloads |
| **Wasabi** | Roughly $0.00699/GB-month; no egress or API fees | Requires 90-day retention on stored data | S3-compatible; 11-nines durability; high performance |
| **Backblaze B2** | $0.006/GB-month | Free egress up to 3× storage per month, then $0.01/GB; API requests: first 2,500 daily downloads free, then $0.004 per 10,000 requests | S3-compatible; 11-nines durability; limited data centers (mainly US/EU) |
| **Amazon S3 (Standard)** | $0.023/GB-month; request charges $0.0004/1,000 PUT and $0.0004/10,000 GET requests | $0.09/GB egress (first 1 GB per month free) | Highly mature; broad regional coverage; comprehensive lifecycle management and object lock |

### Analysis & Recommendation

Given the start-up nature of the project and the need to control costs:

1. **DigitalOcean Spaces** – At $5/month for 250 GB storage and 1 TB egress, Spaces offers the cheapest entry cost with included bandwidth and a built-in CDN. For the MVP (≈200 GB), the monthly cost would be ≈$5–$8. Its S3-compatible API reduces lock-in, and integrated edge caching simplifies delivery. This is a strong choice for launching.

2. **Wasabi or Backblaze B2** – When storage grows beyond Spaces' bundle, these providers offer lower per-GB rates (≈$0.007/GB-month) and free or inexpensive egress. They are good for long-term scaling. Wasabi's 90-day retention might be acceptable since recipe videos are not removed frequently.

3. **Cloudflare R2** – With free egress and integration with Cloudflare's CDN, R2 excels when download volume is high. However, its per-GB storage cost is higher ($0.015/GB), and operations fees may accumulate. Suitable for heavy streaming scenarios or for mixing with Cloudflare's edge functions.

4. **Amazon S3** – S3 is expensive for small budgets ($0.023/GB storage and $0.09/GB egress) but offers the richest feature set (Glacier for archiving, object lock, cross-region replication). It's often used by large enterprises; for this project it may not be cost-effective unless specific AWS integrations are required.

**Recommendation:** Start with DigitalOcean Spaces for its predictable low monthly cost and integrated CDN. To avoid vendor lock-in, store your video metadata in the database and use S3-compatible libraries so migration to Wasabi, Backblaze B2 or Cloudflare R2 is straightforward. As usage grows, you can transfer objects to a cheaper provider by iterating over each file and copying it to the new bucket.

### Video Delivery

1. **Encoding** – When an admin uploads a recipe video, the Rails backend should transcode it into multiple bitrates (e.g., 1080p/720p/480p) and segment it into HLS (.m3u8 playlist with .ts segments). Use FFmpeg in a background job or an external service (e.g., AWS Elastic Transcoder, Cloudinary) to perform transcoding. Store the resulting segments in your object storage bucket.

2. **Signed URLs** – Keep storage buckets private. The backend generates pre-signed URLs for each HLS playlist and segment. A pre-signed URL is a special URL with embedded credentials and an expiry time; it allows the client to access the object without exposing long-term credentials. Amazon S3 objects are private by default; the backend signs the URL using its credentials and sets an expiry, and the client can then use the URL to download the object. This concept also applies to other S3-compatible storage (Wasabi, B2, R2). Pre-signed URLs protect your content because they expire and can be tied to a specific user or IP.

3. **Content Delivery Network (CDN)** – A CDN caches video segments at edge locations near users. CDNs improve load times, reduce bandwidth costs, increase availability and provide DDoS protection. The network shortens the distance between clients and resources by placing servers near internet exchange points. If you use DigitalOcean Spaces or Cloudflare R2, CDN capabilities are built in. For Wasabi or Backblaze B2, integrate with Cloudflare CDN or Fastly to deliver segments.

4. **Caching strategy** – Configure Cache-Control headers on HLS playlists and segments to control how long they are cached by the CDN and clients. Shorter TTL (e.g., 5 minutes) on playlists ensures new segments are discovered, while longer TTL (e.g., 1 day) on segments reduces origin requests. Combine this with Rails' low-level caching to avoid repeated pre-signed URL generation and to store frequently accessed video metadata.

## Scalability & Cost Considerations

- **Storage cost** – Starting with 50 recipes (≈200 GB) and 100 users, the cost on DigitalOcean Spaces is within the base subscription ($5/month). If each recipe grows and more recipes are added, migrating to Wasabi or Backblaze B2 may reduce per-GB cost. For 1 TB of data, Wasabi would cost ≈$7/month vs. $20 on DigitalOcean Spaces. Plan migration scripts to copy objects via S3 API.

- **Bandwidth cost** – DigitalOcean includes 1 TB egress. If monthly video delivery exceeds 1 TB, additional bandwidth is $0.01/GB. At 100 users streaming 10 GB/month each, the application would use ≈1 TB egress and incur minimal overage. For larger scale or heavy streaming, Cloudflare R2's zero-egress may be more economical.

- **Database scaling** – Use managed PostgreSQL with automatic backups, high availability and read replicas. Consider caching read-heavy queries in Redis. Use connection pooling (e.g., puma + connection pool) to handle concurrency. Use environment variables to tune database pool size and worker threads.

- **Background processing** – Offload video encoding and thumbnail generation to background workers and queue systems (Sidekiq). Use separate worker dynos/containers to avoid blocking web requests.

- **Monitoring & Observability** – Deploy monitoring for CPU, memory and database performance. Use logs (lograge) and error tracking (Sentry). Centralize logs using services like Papertrail or LogDNA; unify with analytics events to track user behavior.

## Putting It All Together – Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         React Native Client                              │
│                                                                          │
│  • Sign up / Sign in (API requests)                                     │
│  • Subscription purchase via Stripe                                     │
│  • Request recipe list, video metadata                                  │
│  • Play video via HLS using react-native-video; uses signed URLs        │
│  • Send analytics events to analytics SDK (Firebase/Mixpanel)           │
└──────────────────────────────────────────────────────────────────────────┘
                         ↓ JSON/HTTPS (TLS)
┌──────────────────────────────────────────────────────────────────────────┐
│                      Ruby on Rails API (API-only)                        │
│                                                                          │
│  • Authentication: Devise/built-in generator + bcrypt                   │
│  • Authorization: Pundit or policy objects                              │
│  • Subscription handling: Integrate Stripe Billing API                  │
│  • Admin interface to upload videos, create recipes                     │
│  • Generate pre-signed URLs for HLS playlists and segments              │
│  • Serve JSON data to clients; log events and metrics                   │
│  • Background jobs (Sidekiq) for video encoding/transcoding, email      │
└──────────────────────────────────────────────────────────────────────────┘
                         ↓ S3-compatible API
┌─────────────────────────────┐   ┌─────────────────────────────┐
│   Object Storage Provider   │   │   Managed PostgreSQL DB     │
│   (DigitalOcean Spaces /    │   │   (DO Managed DB / RDS)     │
│    Wasabi / B2 / R2)        │   │                             │
│  • Stores HLS segments &    │   │  • Stores users, recipes,   │
│    playlists                │   │    video metadata, logs     │
│  • Private bucket; only     │   │  • Uses encryption at rest  │
│    accessible via pre-signed│   │  • Automatic backups        │
│    URLs                     │   │  • Read replicas for scale  │
└─────────────────────────────┘   └─────────────────────────────┘
           ↑ CDN (Cloudflare or built-in)
           • Caches HLS segments globally to reduce latency and bandwidth
```

## Development Roadmap

1. **Set up project infrastructure** – Create a Rails API and a React Native app. Configure a PostgreSQL database using a managed service. Commit code to a version control system (GitHub/GitLab) and set up CI/CD pipelines.

2. **Authentication and authorization** – Integrate Devise for user sign-up, sign-in and password reset; implement RBAC for admin vs. subscriber. Add 2FA if necessary.

3. **Payment integration** – Configure Stripe Billing in test mode. Create subscription plans (e.g., monthly, yearly). Implement webhook endpoints to update subscription status in the database and restrict access for non-paying users.

4. **Admin portal** – Use Rails Admin or build custom pages for uploading recipe videos and entering metadata. On upload, trigger a background job to transcode the video into multiple bitrates and HLS segments. Store the segments in object storage and save metadata in the database.

5. **API endpoints** – Implement endpoints for listing recipes, retrieving video metadata, starting a video session and obtaining pre-signed URLs. Ensure controllers authorize the current user before returning video URLs.

6. **Client implementation** – Build React Native screens for browsing recipes, watching videos and managing subscriptions. Use react-native-video to play HLS streams. Integrate the chosen analytics SDK and send events.

7. **Logging and analytics** – Set up a logging service and analytics dashboard. Use Firebase or Mixpanel to track in-app events. Use Stripe analytics for revenue metrics.

8. **Performance tuning** – Configure caching for API responses and view fragments. Tune database queries and indexes. Monitor performance and scale the database or application servers as needed. Use CDN caching for video segments.

9. **Security hardening** – Enforce HTTPS, implement secure session cookies, protect against CSRF/XSS, use Active Record Encryption, and audit dependencies. Perform penetration testing and vulnerability scans.

## Conclusion

The proposed design leverages cost-efficient, scalable components while maintaining flexibility. DigitalOcean Spaces with its $5/month base plan and built-in CDN provides a low-barrier entry point, while S3-compatible APIs ensure that migrating to alternatives like Wasabi, Backblaze B2 or Cloudflare R2 remains straightforward. By employing Rails' authentication generator or Devise for secure user management, Stripe Billing for subscription handling, adaptive streaming via react-native-video and CDN caching, the system can deliver recipe videos securely and efficiently to subscribers while controlling costs and remaining future-proof.

## References

1. Adding Video to Your React Native App with react-native-video | Cloudinary  
   https://cloudinary.com/guides/front-end-development/adding-video-to-your-react-native-app-with-react-native-video

2. Stripe Billing Review 2025: Features and Benefits Explained  
   https://unibee.dev/blog/stripe-billing-review/

3. Top 19 Mobile App Analytics Tools 2025 [Updated]  
   https://uxcam.com/blog/top-10-analytics-tool-for-mobile-in-2018/

4. Securing Rails Applications — Ruby on Rails Guides  
   https://guides.rubyonrails.org/security.html

5. Setting Up A Secure Ruby On Rails Environment On UpCloud: Best Practices - UpCloud  
   https://upcloud.com/resources/tutorials/setting-up-a-secure-ruby-on-rails-environment-on-upcloud-best-practices/

6. Caching with Rails: An Overview — Ruby on Rails Guides  
   https://guides.rubyonrails.org/caching_with_rails.html

7. What is a content delivery network (CDN)? | How do CDNs work? | Cloudflare  
   https://www.cloudflare.com/learning/cdn/what-is-a-cdn/

8. Amazon S3 vs Digital Ocean Spaces for Object Storage in 2025  
   https://www.taloflow.ai/guides/comparisons/amazons3-vs-digitaloceanspaces-object-storage

9. Best Cloud Storage Services 2025: Top 10 Ranked | Galaxy  
   https://www.getgalaxy.io/resources/best-cloud-storage-services-2025

10. Pricing · Cloudflare R2 docs  
    https://developers.cloudflare.com/r2/pricing/

11. 5 Cheap Object Storage Providers  
    https://sliplane.io/blog/5-cheap-object-storage-providers

12. What is the difference between Amazon S3 and Wasabi?  
    https://docs.wasabi.com/v1/docs/what-is-the-difference-between-amazon-s3-and-wasabi

13. Cloudflare R2 vs Backblaze B2 vs Wasabi vs Amazon S3 in 2025: Pricing (Egress/Requests), S3-Compatibility, Immutability/Lifecycle, and the Best Object Storage for VPS Backups - Onidel Cloud  
    https://onidel.com/cloudflare-r2-vs-backblaze-b2/

14. The illustrated guide to S3 pre-signed URLs - fourTheorem  
    https://fourtheorem.com/the-illustrated-guide-to-s3-pre-signed-urls/