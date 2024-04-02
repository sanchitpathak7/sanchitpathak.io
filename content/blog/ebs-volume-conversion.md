+++
title = "EBS Volume GP2 to GP3 Conversion using a CloudWatch Events Triggered Lambda Function"
+++

In this blog post I am going over how to automate the conversion of EBS volumes from the GP2 type to GP3 by using CloudWatch Events to trigger a Lambda function upon volume creation ensuring that new volumes are automatically optimized.

For a 2000 GB volume with approximate usage of 12 hours in a month, the charge calculation would be:
  - For GP3: ($0.08 per GB-month * 2,000 GB * 43,200 seconds / (86,400 seconds/day * 30-day month)) = $2.667
  - For GP2: ($0.10 per GB-month * 2,000 GB * 43,200 seconds / (86,400 seconds/day * 30 day-month)) = $3.33

Reference: [EBS-Pricing](https://aws.amazon.com/ebs/pricing/)

---
...Loading