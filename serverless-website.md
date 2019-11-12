# AWS Serverless Websites

## Purpose

The purpose of this document is to establish a procedure for standing up infrastructure to assist with phishing and malware distribution.  AWS S3 websites are a good fit for standing up phony websites for the purpose of making a domain name look legitimate.  S3 websites are virtually free, you pay a small fee to have your domain managed by Route53 and pay only for storage and transfer of data on AWS S3.  It's possible to run a phony infrastructure for less than $1 per month.

## Installation

### Windows
Download the latest MSI from this https://aws.amazon.com/cli/

### Mac & Linux
Requires Python 2.6.5 or higher.  Install with pip.
```
pip install awscli
```

## Configure
Once you have installed the AWS CLI, you will need credentials, obtain them from your administrator.  Once you have your credentials, issue the command below.  You will need to choose a default region and output format.  If you are unsure, choose "us-east-1" as the region and "json" as the format.

```
aws configure --profile=SOME-UNIQUE-VALUE-HERE
```
Follow the prompts below is a sample of what the AWS credentials should look like.

- aws_access_key_id =AKIAUUVBKJXXXXXXXXXX
- aws_secret_access_key = Q+T6rXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXGN58T

## Preparation

### Domain registration
It is still possible to run a website on S3 if the domain was not registered with AWS.  The downside to external registrations is that you will have to manually configure the DNS verification through your domain registrar.

Another option is to use the AWS name servers.  The --caller-reference field is required and is arbitrary, in this example, I chose to use a date mangle of "yyyyMMddHHmmss", however, you can use any value you want as long as it is unique.
```
aws route53 create-hosted-zone --name my-domain.com --caller-reference 20191110123800 --profile profile_name
```
On linux, you can use the "date" command to automatically generate the caller reference.
```
aws route53 create-hosted-zone --name my-domain.com --caller-reference $(date +%Y%m%d%H%M%S) --profile profile_name
```

Take note of the "NameServers" array near the end of the response, you will need to update your domain's name servers to these values.  These values are not guaranteed to be the same between requests, meaning if you point another domain at AWS, the name servers may be different.
```
{
    "Location": "https://route53.amazonaws.com/2013-04-01/hostedzone/ZXXXXXXXXXXJ6D",
    "HostedZone": {
        "Id": "/hostedzone/ZXXXXXXXXXXJ6D",
        "Name": "my-domain.com.",
        "CallerReference": "20191110123800",
        "Config": {
            "PrivateZone": false
        },
        "ResourceRecordSetCount": 2
    },
    "ChangeInfo": {
        "Id": "/change/C2XXXXXXXXXXZA",
        "Status": "PENDING",
        "SubmittedAt": "2019-11-10T06:38:46.671Z"
    },
    "DelegationSet": {
        "NameServers": [
            "ns-324.awsdns-40.com",
            "ns-624.awsdns-14.net",
            "ns-1106.awsdns-10.org",
            "ns-2009.awsdns-59.co.uk"
        ]
    }
}
```

## Create a S3 website
First, you must choose a "DNS correct" name for your bucket.

- **Good** - my-domain-com
- **Bad** - my-domain_com

My personal preference is to not use dots in the name because it can cause problems when creating SSL certificates.

Once you have chosen a valid bucket name, create the bucket with the following command.
```
aws s3api create-bucket --bucket my-bucket-name --region us-east-1 --profile profile_name
```
Next you have to attach a bucket policy that states all items in this bucker shall be public.  If you don't do this, then you have to set the permissions every time you upload a new file.

The policy looks like this. 

**IMPORTANT - Remember to replace "my-bucket-name" with the actual name of your bucket.**
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PublicReadGetObject",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::my-bucket-name/*"
		}
	]
}
```

```
aws s3api put-bucket-policy --bucket my-bucket-name --policy '{"Version":"2012-10-17","Statement":[{"Sid":"PublicReadGetObject","Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::my-bucket-name/*"}]}' --profile profile_name
```

NOTE: On the Windows command line, you have to wrap the JSON policy in double quotes and escape the double quotes in the json.  You will end up with an atrocious command line string like the one below.  Optionally, you can save the policy to disk and reference it with the ```file:///path/to/file``` syntax.
```
aws s3api put-bucket-policy --bucket my-bucket-name --policy "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"PublicReadGetObject\",\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::my-bucket-name/*\"}]}" --profile profile_name
```
or
```
aws s3api put-bucket-policy --bucket my-bucket-name --policy file:///path/to/policy.json --profile profile_name
```
 Now the bucket is setup, we have to tell AWS that this is going to be a web site.  You can do that with the following command.

 ```
 aws s3 website s3://my-bucket-name --index-document index.html --error-document error.html --profile profile_name
 ```

 The command tells AWS that you want ```index.html``` to be served when no file name is specified and ```error.html``` to be served when a 404 error occurs. Your new website can be found at:

 http://my-bucket-name.s3-website-us-east-1.amazonaws.com/

 The next step is to assign that bucket a domain name of your choosing and a SSL certificate.

 ## Generate a SSL Certificate

 It seams counter-intuitive, but before you can setup a custom domain name, you have to generate the SSL certificate first.  One of the security controls on generating SSL certificates is that you must prove that you are authorized to generate an SSL certificate on a given domain.  The two most popular methods for verifying that you are authorized to act on behalf of a domain are DNS validation and E-Mail validation.  The certifying authority assumes you are authorized to generate an SSL certificate if you are able to control the domain's DNS records or if you are able to receive email from any of the email addresses saved to the domain's registrar record (Admin, Technical, etc). AWS offers these two options as well.

 When requesting a certificate, you can choose to apply SSL coverage for more than one domain name.  In these examples, we will be requesting a certificate for one domain, but you can easily add more domains to your request via the "Subject Alternative Name" (SAN) field.  SAN's are useful if you need to apply SSL coverage to both the .com and .net of a domain name that you control.  SAN should not be used to provide SSL coverage to sub-domains, instead, you should use a wild card certificate by specifying and asterisks as the sub-domain, for example: __*.mydomain.com__.  Wild cards are useful when you need to cover both the "www" and the apex domain, for example:

- www.mydomain.com
- mydomain.com

You can request a new certificate for any domain with the command below.

```
aws acm request-certificate --domain-name mydomain.com --validation-method DNS --profile profile_name
```
The command above will generate a response similar to the one below.

**IMPORTANT - Take note of the "CertificateArn" field, you will need it later.**
```
{
    "CertificateArn": "arn:aws:acm:us-east-1:31XXXXXXXXXX:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf"
}
```

Ok, this is great, but how do I approve the creation of this SSL certificate?  If you chose E-Mail Verification, check your inbox, the verification link should arrive shortly.  If you chose DNS verification, as we did in the example, you must now create some DNS entries.  To retrieve the values for the required DNS entries, use the following command.

```
aws acm describe-certificate --certificate-arn "ARN FROM ABOVE" --profile profile_name
```
```
{
    "Certificate": {
        "CertificateArn": "arn:aws:acm:us-east-1:31XXXXXXXXXX:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf,
        "DomainName": "mydomain.com",
        "SubjectAlternativeNames": [
            "mydomain.com"
        ],
        "DomainValidationOptions": [
            {
                "DomainName": "mydomain.com",
                "ValidationDomain": "mydomain.com",
                "ValidationStatus": "PENDING_VALIDATION",
                "ResourceRecord": {
                    "Name": "_0XXXXXXXXXXeb33f733d2XXXXXXXXXX1.mydomain.com.",
                    "Type": "CNAME",
                    "Value": "_3XXXXXXXXXXd6f2065469XXXXXXXXXX2.kirrbxfjtw.acm-validations.aws."
                },
                "ValidationMethod": "DNS"
            }
        ],
        "Subject": "CN=mydomain.com",
        "Issuer": "Amazon",
        "CreatedAt": 1573420183.0,
        "Status": "PENDING_VALIDATION",
        "KeyAlgorithm": "RSA-2048",
        "SignatureAlgorithm": "SHA256WITHRSA",
        "InUseBy": [],
        "Type": "AMAZON_ISSUED",
        "KeyUsages": [],
        "ExtendedKeyUsages": [],
        "RenewalEligibility": "INELIGIBLE",
        "Options": {
            "CertificateTransparencyLoggingPreference": "ENABLED"
        }
    }
}
```

Take note of the "ResourceRecord", it contains 3 properties, Name, Type and Value.  You will need all these values to create the required DNS record.  If you are copy/pasting these example, don't forget to update the Name, Type and Value with your own values. 

```
{
	"Comment": "",
	"Changes": [
		{
			"Action": "UPSERT",
			"ResourceRecordSet": {
				"Name": "_0XXXXXXXXXXeb33f733d2XXXXXXXXXX1.mydomain.com.",
				"Type": "CNAME",
				"ResourceRecords": [
					{
						"Value": "_3XXXXXXXXXXd6f2065469XXXXXXXXXX2.kirrbxfjtw.acm-validations.aws."
					}
				]
			}
		}
	]
}
```
Before you can add the required DNS entries, you will need to locate the "hosted-zone-id" of your domain.  You can search for your hosted domain with the following command.

```
aws route53 list-hosted-zones --query 'HostedZones[?Name==`my-domain.com.`].{Id:Id}' --profile profile_name
```
```
[
	{
		"Id": "/hostedzone/Z1XXXXXXXXXXCK"
	}
]
```

With the hosted-zone-id from above, you can submit the following command to add the required CNAME.

```
aws route53 change-resource-record-sets --hosted-zone-id "/hostedzone/Z1XXXXXXXXXXCK" --change-batch '{"Comment":"","Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"_0eeb3ed3e99eb33f733d25230575d201.my-domain.com.","Type":"CNAME","TTL":300,"ResourceRecords":[{ "Value":"_3e590902c3ed6f2065469874b4360eb2.kirrbxfjtw.acm-validations.aws."}]}}]}'
```
The response should look something like the value below.
```
{
    "ChangeInfo": {
        "Id": "/change/C1XXXXXXXXXXE8",
        "Status": "PENDING",
        "SubmittedAt": "2019-11-10T21:46:54.925Z",
        "Comment": ""
    }
}
```

Wait a few minutes and AWS should see your DNS changes and approve your certificate.  Once your certificate is approved, you are ready to apply it to your previously created S3 bucket website.

## CloudFront
S3 websites can not support HTTPS traffic.  The work around for this limitation is to create a cloudfront distribution.  Cloudfront is your very own content delivery network and it is basically free, you are only charged for outbound data transfers.  Cloudfront works similar to a reverse proxy and is situated on "edge" locations.  There are hundreds of edge locations all over the world.  The purpose of an edge location is so the traffic generated by your users in India isn't routed all the way back to the US, instead, your India users are routed to the edge location in India.  AWS does all the hard work of ensuring that people visiting your site are routed to the edge location closest to them.  Once an edge location serves a resource such as ```index.html```, it is cached on that edge location.  Any subsequent requests for that resource are not served from the S3 bucket, instead they are served from the edge location's cache.

Cloudfront also does SSL offloading.  When you configure your distribution, you can tell Cloudfront to serve all your content over SSL.  By default, Cloudfront uses a *.cloudfront.net certificate, however, you can configure Cloudfront to use the certificate you created in the previous step instead.

First, you will need the "CertificateArn" you created in the previous step.  If you lost the Arn, you can look it up with the following command.

```
aws acm list-certificates --profile profile_name
```
You will also need to provide AWS with a distribution configuration.  This can be quite lengthy, you can specify the configuration inline like the example below or you can put the configuration in a json file and pass it in via the ```file://``` protocol.  There are several fields that will need to be updated in the example configuration provided below.

- S3 Bucket URL
- CertificateARN
- 

**Example Configuration**
```
{
	"CallerReference": "cf-ref-$(date +%Y%m%d%H%M%S)",
	"Aliases": {
		"Quantity": 1,
		"Items": [
			"*.my-domain.com"
		]
	},
	"DefaultRootObject": "index.html",
	"Origins": {
		"Quantity": 1,
		"Items": [
			{
				"Id": "Custom-my-bucket.s3-website-us-east-1.amazonaws.com",
				"DomainName": "my-bucket.s3-website-us-east-1.amazonaws.com",
				"CustomOriginConfig": {
					"HTTPPort": 80,
					"HTTPSPort": 443,
					"OriginProtocolPolicy": "http-only",
					"OriginSslProtocols": {
						"Quantity": 3,
						"Items": [
							"TLSv1",
							"TLSv1.1",
							"TLSv1.2"
						]
					},
					"OriginReadTimeout": 30,
					"OriginKeepaliveTimeout": 5
				}
			}
		]
	},
	"DefaultCacheBehavior": {
		"TargetOriginId": "Custom-my-bucket.s3-website-us-east-1.amazonaws.com",
		"ForwardedValues": {
			"QueryString": false,
			"Cookies": {
				"Forward": "none"
			}
		},
		"TrustedSigners": {
			"Enabled": false,
			"Quantity": 0
		},
		"ViewerProtocolPolicy": "redirect-to-https",
		"MinTTL": 0
	},
	"CacheBehaviors": {
		"Quantity": 0
	},
	"Comment": "",
	"Logging": {
		"Enabled": false,
		"IncludeCookies": true,
		"Bucket": "",
		"Prefix": ""
	},
	"PriceClass": "PriceClass_All",
	"Enabled": true,
	"ViewerCertificate": {
		"ACMCertificateArn": "arn:aws:acm:us-east-1:31XXXXXXXXXX:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf",
		"SSLSupportMethod": "sni-only",
		"MinimumProtocolVersion": "TLSv1",
		"Certificate": "arn:aws:acm:us-east-1:31XXXXXXXXXX:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf",
		"CertificateSource": "acm"
	}
}
```

```
aws cloudfront create-distribution --distribution-config '{"CallerReference": "cf-ref-'$(date +%Y%m%d%H%M%S)'","Aliases":{"Quantity": 1,"Items":["*.my-domain.com"]},"DefaultRootObject": "index.html","Origins":{"Quantity": 1,"Items":[{"Id": "Custom-my-bucket.s3-website-us-east-1.amazonaws.com","DomainName": "my-bucket.s3-website-us-east-1.amazonaws.com","CustomOriginConfig": {"HTTPPort": 80,"HTTPSPort": 443,"OriginProtocolPolicy": "http-only","OriginSslProtocols": {"Quantity": 3,"Items": ["TLSv1","TLSv1.1","TLSv1.2"]},"OriginReadTimeout": 30,"OriginKeepaliveTimeout": 5}}]},"DefaultCacheBehavior":{"TargetOriginId": "Custom-my-bucket.s3-website-us-east-1.amazonaws.com","ForwardedValues":{"QueryString": false,"Cookies":{"Forward": "none"}},"TrustedSigners":{"Enabled": false,"Quantity": 0},"ViewerProtocolPolicy": "redirect-to-https","MinTTL": 0},"CacheBehaviors":{"Quantity": 0},"Comment": "","Logging":{"Enabled": false,"IncludeCookies": true,"Bucket": "","Prefix": ""},"PriceClass": "PriceClass_All","Enabled": true,"ViewerCertificate":{"ACMCertificateArn": "arn:aws:acm:us-east-1:31XXXXXXXXXX:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf","SSLSupportMethod": "sni-only","MinimumProtocolVersion": "TLSv1","Certificate": "arn:aws:acm:us-east-1:31XXXXXXXXXX:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf","CertificateSource": "acm"}}' --profile profile_name
```

```
{
    "Location": "https://cloudfront.amazonaws.com/2019-03-26/distribution/EPXXXXXXXXXXM",
    "ETag": "E2E5NLLR2RTWQ1",
    "Distribution": {
        "Id": "EPXXXXXXXXXXM",
        "ARN": "arn:aws:cloudfront::3XXXXXXXXXX7:distribution/EPXXXXXXXXXXM",
        "Status": "InProgress",
        "LastModifiedTime": "2019-11-11T16:01:38.470Z",
        "InProgressInvalidationBatches": 0,
        "DomainName": "dXXXXXXXXXXXXl.cloudfront.net",
        "ActiveTrustedSigners": {
            "Enabled": false,
            "Quantity": 0
        },
        "DistributionConfig": {
            "CallerReference": "cf-ref-XXXXXXXXXX",
            "Aliases": {
                "Quantity": 1,
                "Items": [
                    "*.my-domain.com"
                ]
            },
            "DefaultRootObject": "index.html",
            "Origins": {
                "Quantity": 1,
                "Items": [
                    {
                        "Id": "Custom-my-bucket.s3-website-us-east-1.amazonaws.com",
                        "DomainName": "my-bucket.s3-website-us-east-1.amazonaws.com",
                        "OriginPath": "",
                        "CustomHeaders": {
                            "Quantity": 0
                        },
                        "CustomOriginConfig": {
                            "HTTPPort": 80,
                            "HTTPSPort": 443,
                            "OriginProtocolPolicy": "http-only",
                            "OriginSslProtocols": {
                                "Quantity": 3,
                                "Items": [
                                    "TLSv1",
                                    "TLSv1.1",
                                    "TLSv1.2"
                                ]
                            },
                            "OriginReadTimeout": 30,
                            "OriginKeepaliveTimeout": 5
                        }
                    }
                ]
            },
            "OriginGroups": {
                "Quantity": 0
            },
            "DefaultCacheBehavior": {
                "TargetOriginId": "Custom-my-bucket.s3-website-us-east-1.amazonaws.com",
                "ForwardedValues": {
                    "QueryString": false,
                    "Cookies": {
                        "Forward": "none"
                    },
                    "Headers": {
                        "Quantity": 0
                    },
                    "QueryStringCacheKeys": {
                        "Quantity": 0
                    }
                },
                "TrustedSigners": {
                    "Enabled": false,
                    "Quantity": 0
                },
                "ViewerProtocolPolicy": "redirect-to-https",
                "MinTTL": 0,
                "AllowedMethods": {
                    "Quantity": 2,
                    "Items": [
                        "HEAD",
                        "GET"
                    ],
                    "CachedMethods": {
                        "Quantity": 2,
                        "Items": [
                            "HEAD",
                            "GET"
                        ]
                    }
                },
                "SmoothStreaming": false,
                "DefaultTTL": 86400,
                "MaxTTL": 31536000,
                "Compress": false,
                "LambdaFunctionAssociations": {
                    "Quantity": 0
                },
                "FieldLevelEncryptionId": ""
            },
            "CacheBehaviors": {
                "Quantity": 0
            },
            "CustomErrorResponses": {
                "Quantity": 0
            },
            "Comment": "",
            "Logging": {
                "Enabled": false,
                "IncludeCookies": false,
                "Bucket": "",
                "Prefix": ""
            },
            "PriceClass": "PriceClass_All",
            "Enabled": true,
            "ViewerCertificate": {
                "ACMCertificateArn": "arn:aws:acm:us-east-1:3XXXXXXXXXX7:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf",
                "SSLSupportMethod": "sni-only",
                "MinimumProtocolVersion": "TLSv1",
                "Certificate": "arn:aws:acm:us-east-1:3XXXXXXXXXX7:certificate/a4ca88d7-7a5b-4ea8-8de3-5XXXXXXXXXXf",
                "CertificateSource": "acm"
            },
            "Restrictions": {
                "GeoRestriction": {
                    "RestrictionType": "none",
                    "Quantity": 0
                }
            },
            "WebACLId": "",
            "HttpVersion": "http2",
            "IsIPV6Enabled": true
        },
        "AliasICPRecordals": [
            {
                "CNAME": "*.my-domain.com",
                "ICPRecordalStatus": "APPROVED"
            }
        ]
    }
}
```

From that response, you want to grab the "DomainName" field, it should look something like this: ```dXXXXXXXXXXXXl.cloudfront.net```.  Now we want to create a CNAME record for the "www" sub-domain and the apex domain.
```
aws route53 change-resource-record-sets --hosted-zone-id "/hostedzone/ZXXXXXXXXXXNJR" --change-batch '{"Comment": "","Changes": [{"Action": "UPSERT","ResourceRecordSet": {"Name": "www.my-domain.com.com.","Type": "CNAME","TTL": 60,"ResourceRecords": [{"Value": "dXXXXXXXXXXizl.cloudfront.net."}]}},{"Action": "UPSERT","ResourceRecordSet": {"Name": "my-domain.com.com.","Type": "A","AliasTarget": {"HostedZoneId": "Z2FDTNDATAQYW2","DNSName": "dXXXXXXXXXXizl.cloudfront.net","EvaluateTargetHealth": false}}}]}' --profile profile_name
```

Your done.

## Scraping Content

Now that you have an SSL enabled website hosted on S3, what are you going to put there?  Hello World?  Probably not.  You are going to want to publish some realistic content for the industry you are targeting.  One of the easiest way I have found to scrape content is to use ```httrack```. httrack is free, open source and easy to use.  The fastest way to get started is run the command below.

```
httrack https://target-domain.com  -O /tmp/target-domain.com"
```
The above command will copy the contents of ```target-domain.com``` to ```/tmp/target-domain.com```.  But it's rarely that simple, maybe you want to remove certain word or better yet, replace the target company name with your evil company name.  ```httrack``` allows you to execute a shell command after each file is saved.  You can use the ```sed``` command to do string replacements.
```
httrack https://target-domain.com  -O /tmp/target-domain.com" -V "/usr/bin/sed -i 's/acme llc/evil llc/gi' \$0"
```
You can chain expression in ```sed```, just separate them with a semi-colon.
```
httrack https://target-domain.com  -O /tmp/target-domain.com" -V "/usr/bin/sed -i 's/acme llc/evil llc/gi;s/acme-domain/evil-domain/gi' \$0"
```
Sometimes you only want to scrape enough of the site to look legit, but you don't want to scrape the whole site.  Some sites are HUGE and have thousands of pages, you can limit how many links ```httrack``` will follow by using the ```+r``` switch.  In the following example, ```httrack``` will only go six links deep.
```
httrack https://target-domain.com  -O /tmp/target-domain.com" -V "/usr/bin/sed -i 's/acme llc/evil llc/gi;s/acme-domain/evil-domain/gi' \$0" +r6
```
You can get more fancy than that, but the commands above should be enough to get you started.  Now all that is left is to upload the downloaded files to AWS S3.

```
aws s3 cp /tmp/target-domain.com s3://my-bucket/ --recursive --profile profile_name
```