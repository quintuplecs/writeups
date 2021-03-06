# Whistleblow
## CSAW CTF Quals 2020 | Web
#### @jeffreymeng | September 13, 2020

**Problem Statement:**

> One of your coworkers in the cloud security department sent you an
> urgent email, probably about some privacy concerns for your company.

**Hint 1:**

> Presigning is always better than postsigning

**Hint 2:** 

> Isn't one of the pieces you find a folder? Look for flag.txt!

**Attachments:** (1)
File named `letter` with text:

> Hey Fellow Coworker,
> 
> Heard you were coming into the Sacramento office today. I have some
> sensitive information for you to read out about company stored at
> ad586b62e3b5921bd86fe2efa4919208 once you are settled in. Make sure
> you're a valid user! Don't read it all yet since they might be
> watching. Be sure to read it once you are back in Columbus.
> 
> Act quickly! All of this stuff will disappear a week from 19:53:23 on
> September 9th 2020.
> 
> \- Totally Loyal Coworker

One interesting thing about this problem is that no link is given to us. If not for the first hint, this challenge would probably be a lot harder (though the problem statement offers a clue: `cloud security`, which implies aws or some other cloud computing company).

When we google `presigning`, the first (non-dictionary) result is AWS S3. AWS S3 is basically a storage service by amazon that stores your files for you.

Reading a bit about AWS S3 presigning, we learn that it is a way to generate a URL that allows users to make GET requests to retrieve files from a bucket. The link is secured using some url parameters.

At this point, a few things in the message become clear. Sacramento and Columbus likely refer to two different AWS regions (`us-west-1` and `us-east-2`, respectively), because they are the capitals of the states which these regions are based in. The date given in the message (a week from 9/9/20 19:53:23) likely refers to the expiration date of the presigned url. The hex string `ad586b62e3b5921bd86fe2efa4919208` is likely the bucket id.

We don't know what to do with the expiration date yet, but we can first try accessing the bucket. Thus, we make a GET request of the format `https://<bucket-id>.s3.<region-code>.amazonaws.com`. You could use a curl, a REST client, or just enter the URL into your browser.

We first try the region `us-east-2` (there's no particular reson why I chose this over `us-west-1`).
When we load `https://ad586b62e3b5921bd86fe2efa4919208.s3.us-east-2.amazonaws.com/`, we get a response in the form of XML.
```
<Error>
<Code>PermanentRedirect</Code>
<Message>The bucket you are attempting to access must be addressed using the specified endpoint. Please send all future requests to this endpoint.</Message>
<Endpoint>ad586b62e3b5921bd86fe2efa4919208.s3-us-west-1.amazonaws.com</Endpoint>
<Bucket>ad586b62e3b5921bd86fe2efa4919208</Bucket>
<RequestId>8Y4REZ2Y1KDJAY0M</RequestId>
<HostId>b5cOBOl16udqCywyFe9lBMpuO5f2HOuz/qq+0Vr72FfmDRHY8ejzDP/iCdprjZArMWLxs6SgKsA=</HostId>
</Error>
```

This message basically states that we should be using region `us-west-1`. However, GETting `ad586b62e3b5921bd86fe2efa4919208.s3-us-west-1.amazonaws.com` returns an access denied error. At this point, we refer back to the message. Specifically, it mentions `Make sure you're a valid user!`.

Something interesting about AWS S3 is that they have a permission setting that makes the bucket available only to `Authenticated Users`. This seems like it would be more secure, but it actually means *any* user with an account, regardless of whether they are associated with the account that created the s3 bucket.

At this point, it's necessary to use the AWS cli, which can be installed [here](https://aws.amazon.com/cli/).  First, run the command `aws configure` using credentials which you can obtain by logging in to (or creating) an aws account. Create an IAM user and grant it the `AmazonS3FullAccess` permission.

*(It was at this point that I made my first mistake. At first, I didn't realize that I needed to attach the permission, and so when I tried to use the CLI, I was getting `Access Denied` errors, which I thought were related to the challenge itself, but was actually because my credentials didn't have permissions to run s3 commands.)*

After configuring our aws credentials, we can list all the objects inside the AWS s3 bucket with `aws s3 ls s3://ad586b62e3b5921bd86fe2efa4919208`.
This returns a large list of folders with meaningless names. To include the files within the folders, we add the `--recursive` flag.
<details>
  <summary>This returns a large list of text files nested within folders.</summary>

```
2020-09-11 16:24:37         30 06745a2d-18cd-477a-b045-481af29337c7/398c4eb4-f081-4e8b-86ed-5a1e5ddb9de1/a7941336-d167-4855-b36c-208f47418704.txt
2020-09-11 16:24:37         30 06745a2d-18cd-477a-b045-481af29337c7/398c4eb4-f081-4e8b-86ed-5a1e5ddb9de1/b0b92a9c-bcf0-4c2a-9e52-b0635c2915b9.txt
2020-09-11 16:24:37         30 06745a2d-18cd-477a-b045-481af29337c7/6bb1100f-9a7c-471e-a54d-e6edd33c2c1b/2a2f6364-f50c-4cdd-bfa5-fef7b28d6cda.txt
2020-09-11 16:24:37         30 06745a2d-18cd-477a-b045-481af29337c7/6bb1100f-9a7c-471e-a54d-e6edd33c2c1b/55a885b5-ad4f-44d5-9ba1-abaa1a6870d8.txt
2020-09-11 16:24:39         30 06745a2d-18cd-477a-b045-481af29337c7/9ac3b19b-41db-42cf-a7b1-5c074407eaaf/4e0bbf0a-a2ac-40f0-84de-79612bba4c8c.txt
2020-09-11 16:24:39         30 06745a2d-18cd-477a-b045-481af29337c7/9ac3b19b-41db-42cf-a7b1-5c074407eaaf/68432160-a0fb-44a0-bfee-5a203cef701b.txt
2020-09-11 16:24:38         30 06745a2d-18cd-477a-b045-481af29337c7/e45d1d26-7cd7-48cd-aa71-d65cec6cf305/4f9ea583-ffa0-465c-9857-43f6351a0ee9.txt
2020-09-11 16:24:38         30 06745a2d-18cd-477a-b045-481af29337c7/e45d1d26-7cd7-48cd-aa71-d65cec6cf305/aabdae7a-b40e-4d53-86af-9a172296f6fb.txt
2020-09-11 16:24:38         30 06745a2d-18cd-477a-b045-481af29337c7/eea58b6c-f4ba-427e-afe6-353f8bd2dc45/1b8ac827-5415-4cb7-8d9f-8b96f6e2baff.txt
2020-09-11 16:24:38         30 06745a2d-18cd-477a-b045-481af29337c7/eea58b6c-f4ba-427e-afe6-353f8bd2dc45/d446e932-272c-45df-ac1c-c438dbbac0a8.txt
2020-09-11 16:25:01         30 23b699f9-bc97-4704-9013-305fce7c8360/0c8a42bf-99cb-4c13-9689-c140c2101665/2526e808-0ec7-4bc3-9143-60c6d8dac84b.txt
2020-09-11 16:25:01         30 23b699f9-bc97-4704-9013-305fce7c8360/0c8a42bf-99cb-4c13-9689-c140c2101665/7cc7d265-d9db-4023-85d6-a7403833be6d.txt
2020-09-11 16:24:59         30 23b699f9-bc97-4704-9013-305fce7c8360/34428456-7231-4eb0-9ee3-03b0e70a4196/5e5d52d5-da2c-4e56-9e7c-60e44a43b51e.txt
2020-09-11 16:24:59         30 23b699f9-bc97-4704-9013-305fce7c8360/34428456-7231-4eb0-9ee3-03b0e70a4196/e492e6f7-e46f-4df8-ae16-b6e9325571ec.txt
2020-09-11 16:24:59         30 23b699f9-bc97-4704-9013-305fce7c8360/3e470a7c-4f6d-4fd5-8eac-0b88d2529440/52f62de7-96d2-48a1-a74a-f259b8d13e97.txt
2020-09-11 16:24:59         30 23b699f9-bc97-4704-9013-305fce7c8360/3e470a7c-4f6d-4fd5-8eac-0b88d2529440/96ca95d8-b390-48ba-a1e4-6b8138e0002e.txt
2020-09-11 16:25:00         30 23b699f9-bc97-4704-9013-305fce7c8360/4d219046-2441-473f-9d4d-a2e143c46404/97141076-9892-41e7-847b-de924113f72e.txt
2020-09-11 16:25:00         30 23b699f9-bc97-4704-9013-305fce7c8360/4d219046-2441-473f-9d4d-a2e143c46404/b7f8be9e-3b7a-474c-ad1b-8cee070a0c17.txt
2020-09-11 16:25:00         30 23b699f9-bc97-4704-9013-305fce7c8360/d2c7a92c-eb7d-4fbf-afc6-eb715619610e/61d14106-96ea-4963-9b99-67d55860b1f7.txt
2020-09-11 16:25:00         30 23b699f9-bc97-4704-9013-305fce7c8360/d2c7a92c-eb7d-4fbf-afc6-eb715619610e/eaca3e4b-6dab-4a07-b6bb-7e2e0133e44d.txt
2020-09-11 16:24:15         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/049803d7-e25d-4d0e-923a-1f6b8103a833/17818e42-d6db-4ec4-8fdc-db954218c3ae.txt
2020-09-11 16:24:14         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/049803d7-e25d-4d0e-923a-1f6b8103a833/45ba35e5-7a3a-449c-9f11-eeeb73650719.txt
2020-09-11 16:24:14         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/5c6e2384-17b5-45e3-a32c-68d97bb0e2f6/f3188fe0-1762-4e37-89f7-6fa4f816e6af.txt
2020-09-11 16:24:14         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/5c6e2384-17b5-45e3-a32c-68d97bb0e2f6/fe8fa05d-9689-497b-9b26-fb5c3222db57.txt
2020-09-11 16:24:14         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/6664db8b-a451-484d-bc67-995a28c4627d/3643dc5b-4450-4427-8f86-81e2dbe2e868.txt
2020-09-11 16:24:14         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/6664db8b-a451-484d-bc67-995a28c4627d/5b5622f9-d0e4-49a0-82b7-ee63c8f6a3d6.txt
2020-09-11 16:24:12         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/a89378b4-3969-4720-a9ee-eac3782678d9/882340e9-b5f1-4837-af0f-2ba189eb0cd6.txt
2020-09-11 16:24:13         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/a89378b4-3969-4720-a9ee-eac3782678d9/eeb7b68b-8094-4422-ad15-363d1f924c50.txt
2020-09-11 16:24:13         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/b75473f4-d677-4df3-b7d2-cc1b92d9ba1e/4eeb7c79-36aa-4549-b890-bedbd6416af0.txt
2020-09-11 16:24:13         30 24f0f220-1c69-42e8-8c10-cc1d8b8d2a30/b75473f4-d677-4df3-b7d2-cc1b92d9ba1e/8e52e26b-13ec-40bd-9068-cc85563dcff1.txt
2020-09-11 16:24:11         30 286ef40f-cee0-4325-a65e-44d3e99b5498/32e04253-0a77-405a-a18d-fdb660ccb053/9000b94b-4843-4434-9216-24b11c81a196.txt
2020-09-11 16:24:11         30 286ef40f-cee0-4325-a65e-44d3e99b5498/32e04253-0a77-405a-a18d-fdb660ccb053/df347570-b528-4130-92d8-0a2a0d0766ea.txt
2020-09-11 16:24:10         30 286ef40f-cee0-4325-a65e-44d3e99b5498/4c68f540-ea8b-4102-82e6-72f02d79fc2d/c5cc3215-f5cd-4538-a05b-860e4d3f19a0.txt
2020-09-11 16:24:10         30 286ef40f-cee0-4325-a65e-44d3e99b5498/4c68f540-ea8b-4102-82e6-72f02d79fc2d/f9a6c699-a064-4637-b6c3-70d34013d6c6.txt
2020-09-11 16:24:12         30 286ef40f-cee0-4325-a65e-44d3e99b5498/79650afe-dcff-4e04-a1b9-50b61eea5c9a/228dafc5-9997-4ac1-8f62-2f027518cd14.txt
2020-09-11 16:24:12         30 286ef40f-cee0-4325-a65e-44d3e99b5498/79650afe-dcff-4e04-a1b9-50b61eea5c9a/761e7a27-807a-426d-a791-306bba49bd34.txt
2020-09-11 16:24:10         30 286ef40f-cee0-4325-a65e-44d3e99b5498/7ff90324-7b2d-4d61-b84c-0f610986913f/1c8bfa38-48ae-4f1f-8ce3-eb4e7606f970.txt
2020-09-11 16:24:10         30 286ef40f-cee0-4325-a65e-44d3e99b5498/7ff90324-7b2d-4d61-b84c-0f610986913f/6dd92685-b264-40f0-bf3d-14475aee2a65.txt
2020-09-11 16:24:11         30 286ef40f-cee0-4325-a65e-44d3e99b5498/99336fab-ad88-47eb-8501-8608cee06a3c/ac2d991f-d5e3-4409-b130-5511152f9ef8.txt
2020-09-11 16:24:11         30 286ef40f-cee0-4325-a65e-44d3e99b5498/99336fab-ad88-47eb-8501-8608cee06a3c/cf7f6d0f-419a-47ca-a748-736fc4524da2.txt
2020-09-11 16:24:53         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/15fe7012-32e9-42d3-9b81-ce841a40bff3/a3b83277-d70b-4ffb-8f1d-29418d17f391.txt
2020-09-11 16:24:53         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/15fe7012-32e9-42d3-9b81-ce841a40bff3/d2cde031-dbd0-4f6f-9a19-c3bc5af4fe7c.txt
2020-09-11 16:24:55         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/21841889-18e3-4dec-9d65-53f5ad380297/112beee0-8486-4bca-b2f4-762ccb013aa8.txt
2020-09-11 16:24:55         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/21841889-18e3-4dec-9d65-53f5ad380297/3163f1a6-d318-4f3d-90fa-16a1d86d2dd8.txt
2020-09-11 16:24:54         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/945f4fac-497e-485d-b429-610b7aca9748/62a15e59-c68d-4f9b-b383-cd769767166d.txt
2020-09-11 16:24:54         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/945f4fac-497e-485d-b429-610b7aca9748/b4d81d8b-1a6a-4db0-8f20-20ac3ab1889e.txt
2020-09-11 16:24:54         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/9b58c8f4-8287-4fec-b36a-b7e49b890a6a/1b012187-e0b8-4bc5-9bdb-1d96ebc0a93a.txt
2020-09-11 16:24:54         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/9b58c8f4-8287-4fec-b36a-b7e49b890a6a/41949983-69a0-4ccc-9155-d17ff4915210.txt
2020-09-11 16:24:55         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/be73b0fa-27ff-40f5-adbb-6afaec13dd58/7718f95a-4408-4a8f-86ea-0e3640db558a.txt
2020-09-11 16:24:55         30 2967f8c2-4651-4710-bcfb-2f0f70ecea5c/be73b0fa-27ff-40f5-adbb-6afaec13dd58/ebc8093e-3f05-48fd-b893-77a431751cc5.txt
2020-09-11 16:24:51         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/b3bed611-c748-4138-a3f1-fc509933cf02/00537b8e-de52-49ec-a2db-b53cb963bdab.txt
2020-09-11 16:24:52         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/b3bed611-c748-4138-a3f1-fc509933cf02/a13bfd76-ce9e-4643-a3e0-ed33bbfc9f7b.txt
2020-09-11 16:24:53         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/bb8a42cf-fa03-4c92-8296-6f152fe10965/032f931c-4e29-451e-beec-d61dcd589a13.txt
2020-09-11 16:24:52         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/bb8a42cf-fa03-4c92-8296-6f152fe10965/fbc341fd-6e77-4c45-999c-84817e2bd3c2.txt
2020-09-11 16:24:51         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/d343109f-64e4-4bf0-b242-10d9db20360b/77c902f6-5988-47d6-9be2-46f0ab9cbb33.txt
2020-09-11 16:24:51         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/d343109f-64e4-4bf0-b242-10d9db20360b/d5e83be8-ff7f-4ed8-9200-9877640ea75f.txt
2020-09-11 16:24:50         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/dbf6b5d1-ed23-40a5-8b04-86a83987b39a/a248b9c3-b2bd-4e51-97c0-3dcbeea9cb38.txt
2020-09-11 16:24:51         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/dbf6b5d1-ed23-40a5-8b04-86a83987b39a/f8c56e1f-92a9-42f1-ba49-6e276484c4f3.txt
2020-09-11 16:24:52         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/ee301336-5869-4036-8bfe-a711c45e6686/57a4f03e-aed7-4001-bd24-a9a2d9ce47e0.txt
2020-09-11 16:24:52         30 2b4bb8f9-559e-41ed-9f34-88b67e3021c2/ee301336-5869-4036-8bfe-a711c45e6686/a8164c2a-ce17-4cfc-909e-ab1db1d560fb.txt
2020-09-11 16:24:35         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/06747a2a-5d8c-4865-b0ea-a7d4782418de/1324b884-35e5-4abc-b381-6fdd8b995628.txt
2020-09-11 16:24:35         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/06747a2a-5d8c-4865-b0ea-a7d4782418de/e8133096-8355-4493-86a9-b5635600a57b.txt
2020-09-11 16:24:35         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/20d5c0b2-f745-4bdb-980c-eb3d9f1efeeb/63756f50-3141-4c41-add3-60b0df1125c8.txt
2020-09-11 16:24:35         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/20d5c0b2-f745-4bdb-980c-eb3d9f1efeeb/e16c7693-bf45-4e13-9001-864aac04243e.txt
2020-09-11 16:24:36         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/269a072d-344d-4303-87f7-1eebc2cd19ed/20d5c302-0129-4604-bb97-92f0b01b2e87.txt
2020-09-11 16:24:36         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/269a072d-344d-4303-87f7-1eebc2cd19ed/c13020e4-5ab5-403d-9ac4-a913df38b229.txt
2020-09-11 16:24:34         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/8ca8724c-0af3-44b6-acb2-4bb635505385/5244b252-ba11-4f82-b655-c5852b4caede.txt
2020-09-11 16:24:34         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/8ca8724c-0af3-44b6-acb2-4bb635505385/c3323481-7013-425e-9ecf-85b9ccc24c5f.txt
2020-09-11 16:24:34         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/aff40b3e-d691-4438-bcc3-ef32b7e06824/8f275f2f-6196-4c88-8c76-4bea917eef3f.txt
2020-09-11 16:24:34         30 32ff8884-6eb9-4bc5-8108-0e84a761fe2c/aff40b3e-d691-4438-bcc3-ef32b7e06824/b210d177-9ace-44ff-af1f-c236278460ec.txt
2020-09-11 16:24:31         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/3fa52aaa-78ed-4261-8bcc-04fc0b817395/0c604534-3ced-4c63-97be-3610a13f6c5e.txt
2020-09-11 16:24:31         31 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/3fa52aaa-78ed-4261-8bcc-04fc0b817395/4bcd2707-48db-4c04-9ec7-df522de2ccd7.txt
2020-09-11 16:24:33         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/4ffdfc69-7e64-442e-8297-96c8133c3d50/13b3d9a7-fb1d-4dd1-a405-66e728c8f6f0.txt
2020-09-11 16:24:33         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/4ffdfc69-7e64-442e-8297-96c8133c3d50/73859aba-4376-4afa-92b8-57f233b6f497.txt
2020-09-11 16:24:33         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/56fc562d-1763-4cc0-9e1c-8cc60e4088ad/2cf19cc1-687b-441e-bbcd-d1e95c3de73c.txt
2020-09-11 16:24:33         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/56fc562d-1763-4cc0-9e1c-8cc60e4088ad/847aac76-8a1e-498c-934b-477298b4be43.txt
2020-09-11 16:24:32         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/b1c2cce5-8299-4985-9755-039bd814392e/7c6600cc-290e-4101-857e-c4eb5d8c600f.txt
2020-09-11 16:24:32         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/b1c2cce5-8299-4985-9755-039bd814392e/d307337f-5b85-476c-9616-9acafd99658e.txt
2020-09-11 16:24:31         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/f9b6daad-cf9f-4607-99fd-f698dc8254ba/00eb18aa-ba99-4c50-abf7-4d670fcebe19.txt
2020-09-11 16:24:31         30 3a00dd08-541a-4c9f-b85e-ade6839aa4c0/f9b6daad-cf9f-4607-99fd-f698dc8254ba/206791a7-86a1-41ca-9c8d-85f1c9a670a8.txt
2020-09-11 16:24:45         30 465d332a-dd23-459b-a475-26273b4de01c/6a22e4d7-a6ad-48b8-8844-014fae83dfe9/933cb2e0-9af0-4392-8c1c-1018674dba1e.txt
2020-09-11 16:24:45         30 465d332a-dd23-459b-a475-26273b4de01c/6a22e4d7-a6ad-48b8-8844-014fae83dfe9/d5c5d428-b42a-4f70-8e4e-fa419036a72b.txt
2020-09-11 16:24:47         30 465d332a-dd23-459b-a475-26273b4de01c/a359d3c1-4f9b-4b73-9d22-635ea71c0b87/96212037-4398-4748-8371-4ad194ff0a1a.txt
2020-09-11 16:24:47         30 465d332a-dd23-459b-a475-26273b4de01c/a359d3c1-4f9b-4b73-9d22-635ea71c0b87/f96761d4-db10-47b9-b8e4-01880937fb1d.txt
2020-09-11 16:24:46         30 465d332a-dd23-459b-a475-26273b4de01c/e074a69f-611c-4041-9027-94406e56396c/82ffead5-47b9-455e-8f4e-aba6188fecec.txt
2020-09-11 16:24:46         30 465d332a-dd23-459b-a475-26273b4de01c/e074a69f-611c-4041-9027-94406e56396c/d0a31d77-0279-48e1-801f-4be77768b9d8.txt
2020-09-11 16:24:47         30 465d332a-dd23-459b-a475-26273b4de01c/e76ae54c-b803-4225-a5ba-008b31f9f130/42372989-2208-4be5-b1b5-f52630735712.txt
2020-09-11 16:24:47         30 465d332a-dd23-459b-a475-26273b4de01c/e76ae54c-b803-4225-a5ba-008b31f9f130/63c9fcdf-4bba-4d94-9e92-55b0b447070a.txt
2020-09-11 16:24:46         30 465d332a-dd23-459b-a475-26273b4de01c/f7f2103d-1979-42f2-b09e-5c66b619bcc9/19d5b944-35f5-4727-b61d-6fdb12f14f7b.txt
2020-09-11 16:24:45         30 465d332a-dd23-459b-a475-26273b4de01c/f7f2103d-1979-42f2-b09e-5c66b619bcc9/d1c60239-0c22-4a3a-b84c-2eb337634239.txt
2020-09-11 16:24:21         30 64c83ba4-8a37-4db8-b039-11d62d19a136/5c66b679-682f-491d-9a7b-48a441546462/182c855f-d4d6-454e-9d82-ef7274158ffa.txt
2020-09-11 16:24:21         30 64c83ba4-8a37-4db8-b039-11d62d19a136/5c66b679-682f-491d-9a7b-48a441546462/1bff7b54-c79a-40f5-88d2-cb1e94f26747.txt
2020-09-11 16:24:22         30 64c83ba4-8a37-4db8-b039-11d62d19a136/65063325-f4fe-4213-8bb4-9da71aaee3fd/0792e310-c304-402d-a567-91631885f9d7.txt
2020-09-11 16:24:23         30 64c83ba4-8a37-4db8-b039-11d62d19a136/65063325-f4fe-4213-8bb4-9da71aaee3fd/131eb969-fba1-4c77-9d38-8f8336942174.txt
2020-09-11 16:24:20         30 64c83ba4-8a37-4db8-b039-11d62d19a136/7e7299b9-f0b1-4267-a774-080b051108e8/9e7fe775-b2f3-449b-ac5c-5aada22366b9.txt
2020-09-11 16:24:20         30 64c83ba4-8a37-4db8-b039-11d62d19a136/7e7299b9-f0b1-4267-a774-080b051108e8/c52835d5-d9f2-4789-b0c5-8dc42590431f.txt
2020-09-11 16:24:22         30 64c83ba4-8a37-4db8-b039-11d62d19a136/9a9beced-019d-4de4-aecb-4fa263cfe94a/2aa0bb71-1d44-40a8-b245-6dcdb508175c.txt
2020-09-11 16:24:22         30 64c83ba4-8a37-4db8-b039-11d62d19a136/9a9beced-019d-4de4-aecb-4fa263cfe94a/410246aa-93cd-4d4d-9b66-d760c31e2043.txt
2020-09-11 16:24:22         30 64c83ba4-8a37-4db8-b039-11d62d19a136/fcdd47dd-9da4-40cc-a06d-859a64d9369e/216771a4-4a26-4322-90d2-84ffa5105384.txt
2020-09-11 16:24:21         30 64c83ba4-8a37-4db8-b039-11d62d19a136/fcdd47dd-9da4-40cc-a06d-859a64d9369e/84669d40-ff49-4d03-a88b-ad72161a341e.txt
2020-09-11 16:24:23         30 6c748996-e05a-408a-8ed8-925bf01be752/1a4b96b8-c772-46c8-9ca4-d7a6ee86276b/8910b504-3131-408c-9546-ad8b9b0d6f8e.txt
2020-09-11 16:24:23         30 6c748996-e05a-408a-8ed8-925bf01be752/1a4b96b8-c772-46c8-9ca4-d7a6ee86276b/9f4cb1f1-b8ed-4d7f-8ba8-eafc3ef8c233.txt
2020-09-11 16:24:25         30 6c748996-e05a-408a-8ed8-925bf01be752/5ecbe651-a490-4cd8-9f45-5ad930a65c85/4c08958e-db26-4f61-b362-bf358964a873.txt
2020-09-11 16:24:24         30 6c748996-e05a-408a-8ed8-925bf01be752/5ecbe651-a490-4cd8-9f45-5ad930a65c85/7f373a5e-5a16-4bda-931b-98f6086de112.txt
2020-09-11 16:24:23         30 6c748996-e05a-408a-8ed8-925bf01be752/a4bdb585-8403-44fa-8411-1fe071754861/6ff604d4-1d02-4359-9c2e-a949972cdae8.txt
2020-09-11 16:24:23         30 6c748996-e05a-408a-8ed8-925bf01be752/a4bdb585-8403-44fa-8411-1fe071754861/c9da2d51-7a6b-48c9-a26d-a881d105e8c8.txt
2020-09-11 16:24:24         30 6c748996-e05a-408a-8ed8-925bf01be752/c1fe922c-aec8-4908-a97d-398029d39236/646c37cf-a7a1-457f-a09f-84a1d8efd9e8.txt
2020-09-11 16:24:24         64 6c748996-e05a-408a-8ed8-925bf01be752/c1fe922c-aec8-4908-a97d-398029d39236/77010958-c8ed-4a7b-802a-f189d0f76ec0.txt
2020-09-11 16:24:25         30 6c748996-e05a-408a-8ed8-925bf01be752/ca6b4fbe-53a0-4e88-879b-a711854552dd/490c8874-4f96-42ec-80e2-df5bc7518569.txt
2020-09-11 16:24:25         30 6c748996-e05a-408a-8ed8-925bf01be752/ca6b4fbe-53a0-4e88-879b-a711854552dd/734a2537-f50e-4a37-b177-22f10de0eee9.txt
2020-09-11 16:24:56         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/3e44bc53-0367-4471-8473-06411687bf33/0d25d4c4-2aae-405a-a27e-7b6e2a88643d.txt
2020-09-11 16:24:56         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/3e44bc53-0367-4471-8473-06411687bf33/cd05b8b0-770e-4908-be51-1df252ada1f3.txt
2020-09-11 16:24:56         21 7092a3ec-8b3a-4f24-bdbd-23124af06a41/7db7f9b0-ab6a-4605-9fc1-1cc8ba7877a1/1b56b43a-7525-429a-9777-02602b52dc1e.txt
2020-09-11 16:24:56         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/7db7f9b0-ab6a-4605-9fc1-1cc8ba7877a1/424ad0e2-2260-419c-91b0-2c65b7079124.txt
2020-09-11 16:24:57         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/9d5bc557-b3d1-498f-bee2-d5190cb07da2/2e66b0f1-3b50-4f8c-97de-1ce884bc4d3a.txt
2020-09-11 16:24:57         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/9d5bc557-b3d1-498f-bee2-d5190cb07da2/d76823e8-d3e1-4aab-9dd2-edb10e09be4d.txt
2020-09-11 16:24:58         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/d404c8a4-1f35-4253-bd5d-2bd90957d686/11605a66-92fe-4d9e-85ef-16d1772ad4db.txt
2020-09-11 16:24:58         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/d404c8a4-1f35-4253-bd5d-2bd90957d686/1e2a0597-2d78-43fd-a967-4a26fa04d700.txt
2020-09-11 16:24:57         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/efb25af4-57f9-4d3f-8dd2-36040e651e9d/2847112f-f01d-4509-81a4-00b080f0cd9b.txt
2020-09-11 16:24:58         30 7092a3ec-8b3a-4f24-bdbd-23124af06a41/efb25af4-57f9-4d3f-8dd2-36040e651e9d/7f407856-cacf-464d-83f3-23a1b7a82fa0.txt
2020-09-11 16:24:17         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/196c8597-a588-47bf-a007-b8ebc3a74db1/05894a8b-9293-495b-8688-fa05badc6439.txt
2020-09-11 16:24:17         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/196c8597-a588-47bf-a007-b8ebc3a74db1/5dc1d2bb-259e-43c3-88d2-311422d7beb4.txt
2020-09-11 16:24:16         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/741afbbf-c586-490f-9e69-d38204e0083d/1de399d1-45f2-4e4d-8589-c2238f34f6a2.txt
2020-09-11 16:24:16         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/741afbbf-c586-490f-9e69-d38204e0083d/aada5f3a-c3d7-4362-80f3-5f6bcf5a075b.txt
2020-09-11 16:24:16         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/d2f2ee52-c674-4667-82ec-47b5ec2c85fd/a6cf5dda-c7be-434f-9bdb-ed7735853b63.txt
2020-09-11 16:24:16         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/d2f2ee52-c674-4667-82ec-47b5ec2c85fd/af873107-ba10-4bcc-a525-b6cb49e8e968.txt
2020-09-11 16:24:17         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/ebc00dd5-d445-4688-89ea-18f27b53775e/15bab410-6745-429a-b00d-ad03293956b1.txt
2020-09-11 16:24:17         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/ebc00dd5-d445-4688-89ea-18f27b53775e/366a351c-972c-42f3-a029-0324e1bbed74.txt
2020-09-11 16:24:15         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/f3bfbea8-6f29-4f45-953c-c326e7896e8c/a81782c4-7265-47ed-9222-4f4665a4a99b.txt
2020-09-11 16:24:15         30 84874ee9-cee1-4d6b-9d7a-24a9e4f470c8/f3bfbea8-6f29-4f45-953c-c326e7896e8c/c76e9fff-1c9e-4348-be45-11737487c191.txt
2020-09-11 16:24:44         30 95e94188-4dd1-42d8-a627-b5a7ded71372/491beb63-ccbd-4872-b792-b9170a29158e/9fd461f7-f2bd-412a-abef-c953f4271e52.txt
2020-09-11 16:24:45         30 95e94188-4dd1-42d8-a627-b5a7ded71372/491beb63-ccbd-4872-b792-b9170a29158e/c246b08f-fe6b-4ae4-b914-8e8d49ae56e2.txt
2020-09-11 16:24:44         30 95e94188-4dd1-42d8-a627-b5a7ded71372/932e8740-cf5c-42bd-8bd3-426e32340bd0/994c4a36-d908-47f7-ad4c-bf9163e56ab1.txt
2020-09-11 16:24:44         30 95e94188-4dd1-42d8-a627-b5a7ded71372/932e8740-cf5c-42bd-8bd3-426e32340bd0/d430a14c-bb05-4b87-848f-12b585dc71ec.txt
2020-09-11 16:24:42         30 95e94188-4dd1-42d8-a627-b5a7ded71372/9a64cde3-6d7e-41a6-a6d2-884eab1e1ce2/c1cb7d20-d4b5-437f-bd54-185c7af7b571.txt
2020-09-11 16:24:42         30 95e94188-4dd1-42d8-a627-b5a7ded71372/9a64cde3-6d7e-41a6-a6d2-884eab1e1ce2/ea2c63cb-e111-4bd4-9e16-c3065c9de466.txt
2020-09-11 16:24:43         30 95e94188-4dd1-42d8-a627-b5a7ded71372/a907fa15-9696-43e2-b88a-b999c1a523fd/6c4e90d5-e9f5-4ff0-943d-7854c302ee8f.txt
2020-09-11 16:24:43         30 95e94188-4dd1-42d8-a627-b5a7ded71372/a907fa15-9696-43e2-b88a-b999c1a523fd/ac6cc140-4eb1-429f-a524-771098a78f17.txt
2020-09-11 16:24:43         30 95e94188-4dd1-42d8-a627-b5a7ded71372/d74ada22-40df-4674-93fc-67cce87293c6/762cf4e5-8199-4498-be6a-a12dd89076ea.txt
2020-09-11 16:24:43         30 95e94188-4dd1-42d8-a627-b5a7ded71372/d74ada22-40df-4674-93fc-67cce87293c6/fb20cae1-3b70-4514-9732-d1194b7715d6.txt
2020-09-11 16:24:18         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/07310b68-10b5-4eea-b948-16844fd2bc87/845b8577-8b16-4bb7-9838-8a57389fa886.txt
2020-09-11 16:24:17         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/07310b68-10b5-4eea-b948-16844fd2bc87/9238143a-efc6-456d-a062-73a560fb4687.txt
2020-09-11 16:24:20         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/91779d79-57d5-4553-b8f3-cc34eaa5f08e/217699db-032f-4d27-960e-ff35985ddcfa.txt
2020-09-11 16:24:20         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/91779d79-57d5-4553-b8f3-cc34eaa5f08e/bddb97d2-6c45-486a-acda-2f20f1f0a33e.txt
2020-09-11 16:24:18         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/93e53588-d241-4fdd-a12b-032ce57aeb3d/6b3aebf8-d268-4d57-b83e-d5658b094243.txt
2020-09-11 16:24:19         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/93e53588-d241-4fdd-a12b-032ce57aeb3d/bbd1da63-008a-4b02-b1ef-1c7465b3fc9b.txt
2020-09-11 16:24:18         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/b96eb4eb-1664-4105-aa63-f17a9c782d2c/6ac710b7-bf1a-499a-b7e7-73360952a7ec.txt
2020-09-11 16:24:18         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/b96eb4eb-1664-4105-aa63-f17a9c782d2c/98989cc1-2b66-4e7d-9481-27564c6f181b.txt
2020-09-11 16:24:19         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/cd0d253e-96ad-48ec-ad7d-0c8d7be65dc4/0f151c35-044c-4b83-b786-0572e71731f9.txt
2020-09-11 16:24:19         30 a50eb136-de5f-4bb6-94ef-e1ee89c26b05/cd0d253e-96ad-48ec-ad7d-0c8d7be65dc4/e8e89386-166f-4080-9f26-e12269c241c3.txt
2020-09-11 16:24:26         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/6e252943-5407-4da6-afed-84f24c41d019/1ca6d630-2ad3-4af5-ab07-99e2b1c72717.txt
2020-09-11 16:24:26         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/6e252943-5407-4da6-afed-84f24c41d019/c4893a97-bfd6-475c-8a26-fed99a1e617f.txt
2020-09-11 16:24:28         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/941db93d-0328-4ba6-9cff-d29997cdf1a1/ba7e9f27-5e6b-4d17-af9c-607a546b352f.txt
2020-09-11 16:24:28         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/941db93d-0328-4ba6-9cff-d29997cdf1a1/bf7572c5-0cb3-4a23-bfd4-ccf85919db44.txt
2020-09-11 16:24:27         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/be48fc98-d215-4faa-b37c-1cbc32fce5a9/04b57634-0132-45d3-af86-65a1cb70bc4c.txt
2020-09-11 16:24:28         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/be48fc98-d215-4faa-b37c-1cbc32fce5a9/f1d6d1b5-379d-4605-a7e9-5af27fcc2584.txt
2020-09-11 16:24:27         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/d2bbdda4-144d-423d-933b-d3ac1b06bf13/9479b71d-c4be-4ea3-af21-5af9a1dd3808.txt
2020-09-11 16:24:27         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/d2bbdda4-144d-423d-933b-d3ac1b06bf13/abf75b44-0020-4259-b4ee-68ef856a08bf.txt
2020-09-11 16:24:26         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/ff68effa-9b05-42c0-b3b8-614e4695f062/31aed2bf-6265-4689-ad4f-30482347b79c.txt
2020-09-11 16:24:26         30 b2896abb-92e7-4f76-9d8a-5df55b86cfd3/ff68effa-9b05-42c0-b3b8-614e4695f062/a6808f9a-16c4-4f7d-9435-3fef1ace281b.txt
2020-09-11 16:24:40         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/0a8d9411-a575-4a9f-b5b0-93ca9c752ed5/2a064e9d-ae1a-4d69-896e-02d2efe9406b.txt
2020-09-11 16:24:40         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/0a8d9411-a575-4a9f-b5b0-93ca9c752ed5/84888334-fb5d-47da-8f01-122de6a784a2.txt
2020-09-11 16:24:41         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/5d62c352-dcda-4f29-a7d7-8ce7110da17a/50301781-2e8c-425c-9432-1677732d3a0c.txt
2020-09-11 16:24:42         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/5d62c352-dcda-4f29-a7d7-8ce7110da17a/87db4af7-9331-4fce-b58d-712223eca7a2.txt
2020-09-11 16:24:40         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/6a64b2ee-7012-44b1-a9f4-4782c0c813c9/08e3a780-7340-4138-b2e3-43f655ad1be3.txt
2020-09-11 16:24:40         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/6a64b2ee-7012-44b1-a9f4-4782c0c813c9/d5a772af-3954-4083-8cd6-c7fc404c98fb.txt
2020-09-11 16:24:39         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/714614ec-6df7-4167-919b-641f05946e06/15e7811b-95b8-4d96-87a4-9afd8672fe36.txt
2020-09-11 16:24:39         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/714614ec-6df7-4167-919b-641f05946e06/1ef850a5-ff3c-4fce-9347-8dd599409883.txt
2020-09-11 16:24:41         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/9e27a3da-dc0b-452d-82c6-f279fa83876a/56ff732b-82e1-461d-bec2-c571ba2b0f43.txt
2020-09-11 16:24:41         30 c05abd3c-444a-4dc3-9edc-bb22293e1e0f/9e27a3da-dc0b-452d-82c6-f279fa83876a/8a72ca64-ac18-4231-ba31-6b5f58c2b080.txt
2020-09-11 16:25:02         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/20218df4-a994-4633-8db2-663d911b1432/00b322a7-9454-4979-b37f-fe277f7bbc93.txt
2020-09-11 16:25:02         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/20218df4-a994-4633-8db2-663d911b1432/73fa39a1-4087-4e92-bc70-6d66f50fdbb6.txt
2020-09-11 16:25:01         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/2b2ce645-d5cf-4cd8-b687-43233f145a8d/6fb0df79-3846-416a-8aa7-cbe78fe4e1c2.txt
2020-09-11 16:25:02         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/2b2ce645-d5cf-4cd8-b687-43233f145a8d/7624cf15-fa4a-4129-9cfc-85caf0f0565a.txt
2020-09-11 16:25:01         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/b01a2bc4-8ca3-434d-84c4-3bb4c47c0936/9a4e0287-2b55-4103-bc4d-dae2fc5ed930.txt
2020-09-11 16:25:01         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/b01a2bc4-8ca3-434d-84c4-3bb4c47c0936/de5f56d0-1dde-4cc2-ad7b-229d19216013.txt
2020-09-11 16:25:02         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/cb745a17-248d-45bb-b615-945e6d98b5ce/6590f82d-6de4-4560-8277-c0d429ad0f04.txt
2020-09-11 16:25:03         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/cb745a17-248d-45bb-b615-945e6d98b5ce/fd5d1a52-127e-4ab9-8551-ff89bf5c1c68.txt
2020-09-11 16:25:03         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/cf97baac-1bd6-4a6d-b275-ba4ea2dc5fa4/213694c9-4c80-46e0-bac8-ac5d8c0059aa.txt
2020-09-11 16:25:03         30 c172e521-e50d-4e30-864b-f12d72f8bf7a/cf97baac-1bd6-4a6d-b275-ba4ea2dc5fa4/49b9ecbc-6bb4-4863-bb1c-657e6d78c4c4.txt
2020-09-11 16:24:29         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/224602c3-6fb2-4e99-adae-2899685c0af5/238cdee4-fc1f-4e24-a8a2-b305aee7f6a4.txt
2020-09-11 16:24:29         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/224602c3-6fb2-4e99-adae-2899685c0af5/e924fe2e-63a8-45aa-a251-9b62092ebf1a.txt
2020-09-11 16:24:30         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/30712906-5470-4211-92a8-3f3654a09b19/58e4240c-0ac7-4f98-8875-f568c0d3d47c.txt
2020-09-11 16:24:30         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/30712906-5470-4211-92a8-3f3654a09b19/88e9b8e6-7e8d-43b5-bfa0-56c6da9ab76d.txt
2020-09-11 16:24:30         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/95772c6c-df7e-41af-aa1d-19bd323ba36c/f60e40b8-7dbc-4ef9-b76e-94ff1f9f1cd5.txt
2020-09-11 16:24:30         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/95772c6c-df7e-41af-aa1d-19bd323ba36c/fb91d1a7-081e-439c-b314-53f047a9bb11.txt
2020-09-11 16:24:29         20 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/acbad485-dd20-4295-99fa-f45e3d5bdb45/1eaddd5d-fe24-4deb-8e6e-5463f395fa03.txt
2020-09-11 16:24:28         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/acbad485-dd20-4295-99fa-f45e3d5bdb45/e4beef91-ab04-4a57-b518-5392d5d38c4e.txt
2020-09-11 16:24:29         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/fc404211-99bd-429b-bc83-fd4bf9cd0f58/82c64279-d2a3-4abd-a63f-846bb0ad4d4c.txt
2020-09-11 16:24:29         30 c9bf9d72-8f62-4233-9cd6-1a0f8805b0af/fc404211-99bd-429b-bc83-fd4bf9cd0f58/917df540-6e6d-49c8-bc66-4c6bdad80a0b.txt
2020-09-11 16:24:49         30 ff4ad932-5828-496b-abdc-6281600309c6/1308dcc1-1939-4a07-8827-8c3ae5e5a781/5a8a8401-de1f-4ee9-b051-ce955dc8a52b.txt
2020-09-11 16:24:49         30 ff4ad932-5828-496b-abdc-6281600309c6/1308dcc1-1939-4a07-8827-8c3ae5e5a781/8e918e27-5244-4afd-995e-49417c6c18d7.txt
2020-09-11 16:24:50         30 ff4ad932-5828-496b-abdc-6281600309c6/16f4b8c2-7b3a-4825-95a3-36b0c417be5f/2ce0596f-5340-47eb-bcce-b05993b49fed.txt
2020-09-11 16:24:50         30 ff4ad932-5828-496b-abdc-6281600309c6/16f4b8c2-7b3a-4825-95a3-36b0c417be5f/bad464bd-60ac-4985-88e6-bdef0be3110d.txt
2020-09-11 16:24:48         30 ff4ad932-5828-496b-abdc-6281600309c6/4fa6601c-0cfb-40b4-b9ef-56ba0d315897/b50f3fdc-2857-4423-8e56-8a4e5131a0a4.txt
2020-09-11 16:24:48         30 ff4ad932-5828-496b-abdc-6281600309c6/4fa6601c-0cfb-40b4-b9ef-56ba0d315897/ce2cc60a-09e1-4a8d-b543-357fa7153c6b.txt
2020-09-11 16:24:49         30 ff4ad932-5828-496b-abdc-6281600309c6/7ce68465-dfd0-4538-b127-9fb0041291f5/6e7db85c-19a4-4b0e-85e9-6ba388aa1c25.txt
2020-09-11 16:24:48         30 ff4ad932-5828-496b-abdc-6281600309c6/7ce68465-dfd0-4538-b127-9fb0041291f5/a84d9b1c-e609-42b8-b12d-d44b1fc3cc18.txt
2020-09-11 16:24:49         30 ff4ad932-5828-496b-abdc-6281600309c6/95af4394-b823-4507-be49-885f971f8836/5fed4838-9c2c-49a7-8903-21ead9df2acb.txt
2020-09-11 16:24:50         30 ff4ad932-5828-496b-abdc-6281600309c6/95af4394-b823-4507-be49-885f971f8836/97e85c72-0733-45ab-9a97-3030355ad4be.txt
```
</details>

It's probably infeasible to look inside all of these files manually, so instead we can copy them all to our computer using `aws s3 cp s3://ad586b62e3b5921bd86fe2efa4919208 ./ctfawsfiles/ --recursive`, and then concatenate all of these files, while preserving a newline, with `wk 1 .ctfawsfiles/**/*.txt`. This gives us all of the file contents without us needing to manually open each file.
<details>
  <summary>Basically, most of the files contain random strings of equal length, but four files stand out. </summary>
  
```python
noezdntlakykwxzziydwxqyzddcjap
nxixpivfmkexnyreylqdkkxnpoavvs
qhkvovdjxuueppwqqerbdvomfbtalw
nlduaqausjuxepoaomxpvbfxrlxpfy
ejoixnpvtuvaqrasvngghfokoquuzo
fnekrowhsilffwoqxzrqiiswteyzwc
clncahbmlvqtxmrfpxmadisczqdgxx
vyiyifgquhhglzgblreruplnigkzrc
bdmerjzmculnkafydhlcjhzzoliszt
qhieneiinswhtfdiuanzadgxpmicqn
dohxhilesuevzfdxjenlerrqxofaer
rjwxsskfhxizposcrlxiqnlrklvwtm
rnqpaendpffrsebyadvvqdlbpsxofh
nsvswtoroskgbwugwugnkffitjvboz
erattxchrjvbhwervtlctqhzbhhiht
vodoujfcbyejgudrkajsjzsswrrjrf
kevtgpqdjxekmqsfubvhrmvipzlziu
yknoxhxfintzwgpearmrmqpihmhitu
zaukkiovenuwfjgjwxbzhxerirjrwf
zrwljyjohptwbevnpneshzsmrhxllo
czbrgdcfmqckzhxowisrduykszgasc
tkuhvajvmgglqgbmwboiwtckablpsj
lwxsnezoxyryphufrremqjgugvhbwe
dmkjloienvghhoeyhkffazexuwaelb
btvssvngjejdgzbrlrqfakbqhpczsr
pxruqfvtivcmphfklhgowllaxzktuh
bvjitotgnqqfejsmmacxjrfgjhitts
ffixkengprbrfyaibsrvhglrfloqjz
qjwmqehtefctpqrgifiyifkaocpyys
vfnpbckvpqyyvxiamigsaxzzbwfasi
gvrzuizevhlakmesvcwlbmkemjoqcp
qsudcyaejkutclnllggwudeqockqvd
emhhfpouuwqqbtdeafglauvvrhktrh
dcfosoxfoxtqcqcrdgvwaqvfzunsig
zohlhlxvuxmyscpoawcinsrplkynwj
ctnybfexvylzdwlauiezxyeoijpryw
zcwpxknpulaxazpgctpzvetksszonu
xhqjfpvxhpisrcmrcpzuvbkamsdnko
ezqyenhrkfyllswkcdwdqunlddtuqb
ojsstcdbqaftqsbjkggrofyzbibkld
ownpxoghloacklignbaxzlqkbkmchu
syhejxepjdcwckbtbhnjmptcntuoec
hnempcjeofecibmkmeikftgjtazeue
anazlqsdfbwckkeyzvylgkjeodfwrm
qhevchbmvvdcsgxpwabzpmzqjkruwp
dxmwouqaahrxjkimcmkybgyiaisfkf
clelmqbejwwmvtlwgzkcmvkaonlhkr
jafgtmusccfuxqwgcaghraipbrwiqe
jiikaswncyaagxudkvyqxxirgvnwgk
hiojvmdfbhjfdldhkecebehmmhpwid
sdszueclvjimmevztenczesscrjopz
mxnbokobuvrzhokbkwzynhpyzreaox
nblmrtkyvsyptodlulcbarrkrcpguc
oxmbfqnqbxeqhcggmcssdchihzbejj
ehdsoxtnhrqtvauxgnojvubrlfuuhn
usvkipfxnjomjknkgqovvmkhlsqcer
rnpveosfybhbcobmdmrihthzriwvew
rzbbjtaggylrhprlgmpwjqraxcnbws
zdkxdjnpcmeargrvrgbgybdnppffnl
pjuzmdvymryoxflsymgoqwvtojvemk
dnjnsaxgumarlysplpstfvnlictavt
yvtgindvtjvzfdmyuybrcnvtkrwcas
ihefmhdlfiucrfudvlnbbfpasaowdh
yxhlbkfwkzuadqhnqqxibuesjstvqw
vkcuxorwzagbmwyxwixkrsrbczsamm
xsddyqnvvesfxfhedvbpuztdhcurtv
axgzhjbunxwwymninmhbgsnxnyhpcz
attqgihispchcgziwpbfoelrfvqouh
xzwdymylwanliwdxndkzcsfnteykkq
nmjzggrszodkzsnoirefqofwpkwzuk
lzcnciddiarslbloyakbzsaztninxu
s3://super-top-secret-dont-look
aokjaxoptweihlbimervwzwqwfzprb
bducoqkmzytksdyyyqezsrxzzmnwuk
jtahqgygrbguvbqjoasrveqxpvgwur
rrnipexnpbqyjpddnffmlnxpxowsuc
rgtjihtxuabvlawanvrjaxtivgcvol
lifyjxqnbnriuzwgiqponimwtatmem
gmlchvcywzlhunkzlkskmkxpatdcgx
ixhdcmsefidqiaydmhxjxhysqcgtfd
gmuitaikficmryeoawzrzzyqauuqlp
ykcjxffvlfqsaaqxsbtjarikctzqac
oljwdnsrahqtxnlllchnrziopaxzdv
pkimoidzytmrqjmzyoikriimvfprdw
nlwxaofgpihgpdpfrilreszhpbhwwj
ffbuyankekfyfekmnjhqpwjqhdrepi
ttbtnodiqhrfqkfpopnpnjydpwkijr
oudhmtzmubotpolgafwivddaahqpqb
ojhabqnshkyicqzugglzevlojtcgew
wlulkqhwlgkdpbrmhdrzbxezfoukst
draqstjjujpavznyaqcdyxinsetppb
anyavvzpiryktposujwpzcxpdxeaey
rkuolzifvvnasykhrbdmdrlvszwrea
wtoksvkjqrugokcvksqoffbythbwrq
gamybugcthiwfqkioxykefxflxohhj
llaagkyaeylsqltgmcclfipxdsdtab
mftmbmixjudxyqsgbvhgmngjpicanw
yifnarrtxzblppttlupjndbwsdvsky
gzwwoputndhenzdskfcxbumayupznl
ewfgnuydxiluqsmslooyhmwwmcmptu
pfamzofqeujwycpakyeaudvxjwuswh
kxqlmhhczgaxfjvdpuogtggmpnyvid
xghclbvaieudlnsvyexglacsyintjr
jkvmuxejuemtebpqkdfdikxzftblgm
exjkkyhjooockdgbdsqkhnimvwwwph
esemixusixlpenyyxudlkybsqbrcht
zhemsfpxxgwkwcaxqwgpgcyhpkongx
3560cef4b02815e7c5f95f1351c1146c8eeeb7ae0aff0adc5c106f6488db5b6b
tenmptzviejfbeygbtnuivreykuzpv
oeuwhbxkosvxiiipercjvzkwyrogtw
sactbjnnnhcqwngiijdezxzbwhuier
oynmdzqryqefmrflewixsmshnnybkf
.sorry/.for/.nothing/
rpozufzmbiwldpchlzfpuxslgcbpit
qccfzdcrrgpdjyczqrbqtsnhecmhwd
ykbrqspuckrepvesgtpaormwosdgob
luunslrqnzzumjxcavolbujxgpbxdd
fvfgmucrdqzdowtbupwzcrlzrhyape
itwrgxzqfpnkonvalrrodkkcjrlpuz
hthnztapgkgykgbllmghqaalnkfefy
cvrravxweiifhvbqvfxfotxtkxedun
otyudmzesuerpxwmqrdyscojbehmfm
nxdobtblpsgeltrauhaphvubjbopwo
gjomeuzeqrkboasdhvyovyaowfjwvq
gpwtuxajiqblcyhpvzzllkyzdcgddg
dglaedmyedjazwldpypeyhafhhbmpx
jzoshwtfgvxlogcavpbqrtpqbfjsxw
lujaeoqqnnjkqsiardhnyarmuuxksa
zqjnabgiubggfwyaxiowtyeorfpkxb
ufpatduwefyzvybykyntptqzjyfwsn
vttxbatyrbakfzczmkjjucxitnucja
fxouprmrkkbemkjdkuogayvfgmnyhw
tjdtvbavbbwegegjkcslwwvmuykunc
vecovxvxuaqzkiahzruxsjzqzpoatb
mrxaexypzjgiymvczqzocarpgbymxe
mikedwybeqxygxegpemlkuaidgpffj
xlaggfrbogpkbarcaebbaqfyxipwfq
fzhuiqxtnkulgkkizwefqporarzdge
jaunxwynwyhduiuyiyzhogmvvtyisj
udrrusueqofmwsfuwfvxyzeziwcaea
kmnattbixamwrafwxzgfaftusvjwoc
vmpunvkytsgxekgghgueczzqvunypt
gyinnclqckntrmzmgvvzymwahmeesm
kwtkbbrvomrhckcyxlwrexkcwkicol
vtedpnsyduehkoewdfusqkxfytszww
jbjtvfchfqytsqhctaaprgckaxdsnx
ofijloilprysicewecdhxrbbskwtnv
fgudfrethxybqvwcszigqvyagxmpag
tbliojbvgmyjojborseyrlenbxtaoy
hvroecpdkezjbtdbfwqqkjcwkrwzep
chmpoegomzzyrbxvgofmwihatepmwz
yzwnhzkkiwzeafihpuukucuudqsbzh
mpzmlputugilrjotmmnbbtrbxhzfpo
qmgwcdafcwatsrnghcnujjbsdahmno
urnvjfmrxoutvgnoycfqcaudrszgbd
evcpicajxjlnocbynjvrcjqgpqaajx
prkgppcgwbjacoumumgxzcokhwheoj
gqtskrdlxlcmkzupgdnhnykhrouchk
yrtmjyxjhwhpoanqnobykavgttmilf
fhslkpdpnmgcszgxddjafdjnxrrozt
jljegkvvuzinhdwbwmuaumafgkslsr
zavgwohffprmutrstrqtptqlqmettt
mycbcxqadkdkvritnnpsxgrzaqtpzc
vnpaukearbizklwzsxfyxcxpiwfrzh
wiuscgowcummedyxxtsycsybvwmkim
kltavalrtrvxcqwmeajrmsfwprhupa
kkuvncccgfiotpjycwhgkngukwahbu
wyaqtyiobwqscwavmhweddeuvvwadi
mbumjcsjxaakycsdpnzvxmewktsjun
znoupwajtduuygkanqvdaruislcoip
icpoqzvwnsywykdpllncmztqjaiwzz
bkbjhmbpltnebosyjferkxfpdcvfze
qqveizltjqvbrucekahzdzsiodwfiy
hafqxumpcywjfvlvdeujyewzlsaqrh
lmhhcygfiocsvzwbuwappvahuksyjt
brfhibeooosokwtsfnnahvdzyymfoj
cydyrgvxlbeeowwevpvtrgpilwlund
jwhwhkyxnweiqbkdbwnnhgmppranwp
kmuxtgaktfsoxbmuwooodldmmaezzm
asuzlhjzqkojpvxijvhrsmqivnhluf
mboudhpxuuwdujxctbhyqwguvoxyke
nievhqpdbpngzfrhsoytpltndaxmal
grikrkcamlruejktrsirurwrbsftax
yvixrdwvttnmoaqsqoeeqxvsiyizym
xybutjfsdtdwjbbppvepjcplamdbns
awxurmlfdrbaauvtkpupshnanvhtxa
AKIAQHTF3NZUTQBCUQCK
dxvyjoizdgdechtnveyxzdmtugmbrg
owoktqxxzflcomrojojsutaulgbjvg
esosgetayxkamrnooknchwlwlypkle
bhwlrskzjzpsiemmsxjktcwcuthamh
vuasnszlnwrzqzygwaedkqwuemgiot
zzhwtaklocactcwadzchhmlcyipepn
rtdoigufmnmycyyzbjxjhcesdoyvri
kcojzcxxjlkbhchouyetqfjivqmtlf
amyidqghgojfsmihwmqmfbxywwuxoi
qiogeuymuabjeddblnxlbywerwfsto
pjstjqidxtjgbzwvrspsmagxbreqcx
hiupkyaydqfjmdtsutriuovvmdfgfd
trhfjmlnaopwcmwvgbiecagfyaybao
```
</details>

The interesting lines are
```
s3://super-top-secret-dont-look
3560cef4b02815e7c5f95f1351c1146c8eeeb7ae0aff0adc5c106f6488db5b6b
.sorry/.for/.nothing/
AKIAQHTF3NZUTQBCUQCK
```
These look like parameters for AWS pre-signed urls. This [aws documentation page](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html#query-string-auth-v4-signing-example) tells us what we need, and how, to create a presigned url.
The first line is a bucket ID, likely containing the flag. 
The second one looks like an SHA hash, likely a signature for the presigned url.
The third one is a folder path, but based on it's name, it also could be nothing.
The fourth one is aws access key ID.

Referring to the second hint, it mentions that one of the pieces that we get is a folder. This is probably `.sorry/.for/.nothing/`, and it mentions to look for flag.txt. Thus, the flag is probably at <bucket id>/.sorry/.for/.nothing/flag.txt

This contains four prices of information that we'll likely need to create a presigned url: bucket id, signature, key id, and path to the flag.

However, we're still missing the Date and expiration values. The text file mentions that we will have a week from `19:53:23 on September 9th 2020`. This likely means that the presigned url (and thus the date parameter), was created on `19:53:23 on September 9th 2020`, and the url is valid for a week.

The [aws documentation page (same as above)](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html#query-string-auth-v4-signing-example)  handily tells us how to construct our URL.

Path: https://super-top-secret-dont-look.s3.us-east-2.amazonaws.com/.sorry/.for/.nothing/flag.txt

X-Amz-Algorithm=AWS4-HMAC-SHA256

X-Amz-Credential=AKIAQHTF3NZUTQBCUQCK/20200909/us-east-2/s3/aws4_request 
*(this is created using the key id, the date from the text file, and the region it tells us to access it from -- columbus i.e. `us-east-2`)*

X-Amz-Date=20200909T195323Z
*(This is just the formatted version of the date above)*

X-Amz-Expires=604800
*(One week in seconds)*

X-Amz-SignedHeaders=host

X-Amz-Signature=3560cef4b02815e7c5f95f1351c1146c8eeeb7ae0aff0adc5c106f6488db5b6b

Thus, our final url is https://super-top-secret-dont-look.s3.us-east-2.amazonaws.com/.sorry/.for/.nothing/flag.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAQHTF3NZUTQBCUQCK/20200909/us-east-2/s3/aws4_request&X-Amz-Date=20200909T195323Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=3560cef4b02815e7c5f95f1351c1146c8eeeb7ae0aff0adc5c106f6488db5b6b.


This is just a text file with one line, the flag.
```
flag{pwn3d_th3_buck3ts}
```
