## AWS Regions
- AWS has regions all around the world - `us-east-1`, `eu-west-3` etc.
- A region is a cluster of data centers.
- Most AWS services are region-scoped.

### Choosing an AWS Region
- Compliance with data governance and legal requirements - data nevers leaves a region without your explicit permission
- Proximity to customers - reduced latency
- Available services within a region - new services and new features aren't available in every region
- Pricing - varies region to region and is transparent in the service pricing page

### Availability Zones
- Each region has many availability zones - usually 3. Min is 3 and max is 6.
- Each availability zone (AZ) is one or more discrete data centers with redundant power, networking and connectivity.
- They are separate from each other, so that they're isolated from disasters.
- They are connected with each other with high bandwidth, ultra-low latency networking forming a region together.

> [!NOTE]
> If a service is available in all the regions, the region will be automatically set to `Global` when we open the service page.

> [!TIP]
> To check which services are available in which regions - [AWS Regional Services](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)
