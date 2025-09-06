<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/6be50800-be2b-464b-86cd-fb981e4991cf" />

---
# What is AWS CloudFront?

Amazon CloudFront is AWS’s Content Delivery Network (CDN) service.
It helps deliver your website, videos, APIs, or apps faster and more securely to users across the world.

Think of it like Amazon’s own global delivery system, where your content is stored in multiple edge locations (servers placed worldwide).

## How CloudFront Works ?

## ⚡ AWS CloudFront Workflow

1. User Request (Viewer) : A user (say in Europe or Asia) requests your website → https://saimestore.com/product.jpg.

2. DNS Resolution : Request first goes to Amazon Route 53 (DNS) (or any DNS). DNS routes user to the nearest CloudFront Edge Location.

3. Edge Location Check : CloudFront checks if the requested file (image, CSS, video, API response, etc.) is cached at the edge location.
   - ✅ If cached (hit) → returns instantly.

   - ❌ If not cached (miss) → request goes to Regional Edge Cache (REC).

4. Regional Edge Cache (REC) : Bigger cache layer between Edge Locations and Origin.
   - If content is present → delivered.
   - If not → forward to Origin Server.

5. Origin Fetch:
   Origin can be:
   - Amazon S3 (static files),
   - EC2 / Load Balancer (dynamic apps),
   - Custom Origin (On-prem, other cloud).
   - CloudFront fetches fresh content from origin.

6. Caching + Delivery :
   - CloudFront caches the content at both:
   - Regional Edge Cache
   - Edge Location nearest to user
   - Then delivers it to the user.

7. Next Requests :
  - Future requests from nearby users get the content directly from the edge cache → lightning fast ⚡.

### 👉 In short:
User → DNS → Nearest Edge → Regional Edge Cache → Origin → Back through Edge → User 
