---
slug: preventing-public-access-for-gcs-buckets
title: Preventing Public Access for GCS Buckets
author: Jeffrey Aven
author_title: Cloud Consultant
author_url: https://github.com/infraql
author_image_url: https://s.gravatar.com/avatar/f96573d092470c74be233e1dded5376f?s=80
image: https://storage.googleapis.com/infraql-web-assets/blog/infraql-gcs-public-access-prevention.png
description: This article shows how to enable public access prevention on Google Cloud Storage (GCS) buckets using InfraQL - a SQL based approach to deploying and querying cloud assets.
keywords: [infraql, google cloud, GCP, infracoding, IaC, infrastructure as code, google cloud storage, cloud storage, GCS, cloud security, CSPM]
tags: [infraql, google cloud, GCP, infracoding, IaC, infrastructure as code, google cloud storage, cloud storage, GCS, cloud security, CSPM]
---

Its easy enough for anyone to deploy a Cloud Storage bucket in google, this can be done through the console, `gcloud`, `terraform` or `infraql` as shown here: [__Deploying and Querying GCS Buckets using InfraQL__](/blog/deploying-and-querying-gcs-buckets-using-infraql).  It is also easy to inadvertently allow users to set public ACLs on a bucket, therefore making its contents publicly visible by default.  There is an easy way to prevent this from happening by [Using public access prevention](https://cloud.google.com/storage/docs/using-public-access-prevention).

Let's work through a real life scenario using InfraQL.

## Step 1 : Run a query to find buckets which do not have public access prevention enforced

Run the following InfraQL query from the `shell` or via `exec`:

```jsx
SELECT name, 
JSON_EXTRACT(iamConfiguration, '$.publicAccessPrevention') as publicAccessPrevention
FROM  google.storage.buckets
WHERE project = 'myco-terraform';
/* returns
|-------------------|------------------------|
|       name        | publicAccessPrevention |
|-------------------|------------------------|
| myco-tf-nonprod   | unspecified            |
|-------------------|------------------------|
| myco-tf-prod      | enforced               |
|-------------------|------------------------|
*/
```

We can see from the query results that the `myco-tf-nonprod` bucket does not have public access prevention enforced, lets fix it...using InfraQL.

## Step 2 : Configure public access prevention for a bucket

Run the following InfraQL procedure to enforce public access prevention:

```jsx
EXEC google.storage.buckets.patch 
@bucket = 'myco-tf-nonprod'
@@json = '{
    "iamConfiguration": {
      "publicAccessPrevention": "enforced"
    }
}';
```

## Step 3: Confirm public access prevention is enforced

Run the first query again, and you should see that the desired result is in place.

```jsx
SELECT name, 
JSON_EXTRACT(iamConfiguration, '$.publicAccessPrevention') as publicAccessPrevention
FROM  google.storage.buckets
WHERE project = 'myco-terraform';
/* returns
|-------------------|------------------------|
|       name        | publicAccessPrevention |
|-------------------|------------------------|
| myco-tf-nonprod   | enforced               |
|-------------------|------------------------|
| myco-tf-prod      | enforced               |
|-------------------|------------------------|
*/
```

Easy!