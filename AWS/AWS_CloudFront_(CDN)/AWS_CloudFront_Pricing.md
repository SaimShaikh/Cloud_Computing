# 📦 AWS CloudFront Pricing Overview

## 1. Free Tier
You get some solid freebies every month as part of AWS’s always-free tier:  
- **1 TB** of Data Transfer Out (DTO) to the internet  
- **10 million** HTTP/HTTPS requests  
- **2 million** CloudFront Function invocations  
- **Free SSL certificates**  
- Full access to all features  

---

## 2. Data Transfer Out to Users
Billed for data sent from edge locations to users (tiered, region-dependent):

| **Region** | **First 10 TB** | **Next 40 TB** | **100–150 TB** | **Beyond** |
|------------|------------------|----------------|----------------|------------|
| US, Europe, Canada | ~$0.085/GB | ~$0.080 | ~$0.060 | Drops further |
| Asia Pacific (incl. India) | ~$0.12/GB | ~$0.105–0.11 | ~$0.09 | Continues decreasing |

---

## 3. Request Pricing (HTTP/HTTPS)
Charged per 10,000 requests:

- **US/EU/Canada**  
  - HTTP: ~$0.0075  
  - HTTPS: ~$0.01  

- **Asia Pacific**  
  - HTTP: ~$0.009  
  - HTTPS: ~$0.012  

---

## 4. Optional Add-On Features
- **Field-Level Encryption**: $0.02 per 10,000 requests  
- **Real-Time Logs**: $0.01 per 1,000,000 log lines  
- **Cache Invalidation**: First 1,000 paths/month free; then ~$0.005 per invalidation  
- **Lambda@Edge**: ~$0.60 per 1M invocations + $0.00005 per GB-second compute  
- **CloudFront Functions**: ~$0.10 per 1M invocations  

---

## 5. Example: Estimated Monthly Bill
**Scenario**: Delivering to North America & Europe (Price Class 100)  

- **10 TB data transfer** @ $0.085/GB → **$850**  
- **30 million HTTPS requests** @ $0.01 per 10K → **$300**  
- **Real-Time Logs** (5 million lines) → $0.05  

**Total ≈ $1,150**

---

## 6. TL;DR Summary

| **Cost Component**            | **US/EU**                  | **Asia / India**            |
|--------------------------------|-----------------------------|-----------------------------|
| First 10 TB Data Transfer      | $0.085 per GB               | ~$0.12 per GB               |
| HTTPS Requests (per 10K)       | ~$0.01                      | ~$0.012                     |
| Cache Invalidation (>1K paths) | ~$0.005 per path            | Same                        |
| Real-Time Logs                 | $0.01 per 1M lines          | Same                        |
| Lambda@Edge                    | $0.60 per 1M invocations    | Same                        |
| CloudFront Functions           | $0.10 per 1M invocations    | Same                        |

---

## 7. Cost-Saving Hacks
- Use proper **cache TTLs** to reduce origin hits  
- Enable **Origin Shield** to cut redundant fetches  
- Pick a **Pricing Class** that fits your region (e.g., 100 for NA/EU)  
- **Batch requests** or consolidate calls to trim HTTP charges  
- Use **stale-while-revalidate** for freshness + cache efficiency  

