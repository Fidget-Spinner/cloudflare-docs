---
title: Certificate Signing Requests
weight: 50
---

For those customers that prefer to acquire their own SSL certificate from a Certificate Authority (CA), Cloudflare can generate the requisite Certificate Signing Request (CSR) with  the customer’s organization name, location, etc.  (The associated private key will be generated by Cloudflare and never leaves our network, avoiding the risk of unsafe handling by you or your customer.)

Once the CSR has been generated, it should be provided to your customer. Your customer will then pass it along to their preferred CA to obtain a certificate and return it to you. After you receive the certificate you should upload it to Cloudflare, referencing the unique CSR ID that was provided to you during CSR creation.

## Generating the private key and CSR

### 1. Build the Certificate Signing Request (CSR) request payload

All fields except for organizational_unit and key_type are required. If you do not specify a `key_type`, the default of `rsa2048` (RSA 2048 bit) will be used; the other option is `p256v1` (NIST P-256).

Common names are restricted to 64 characters, per [RFC 5280](https://tools.ietf.org/html/rfc5280), while subject alternative names (SANs) can be up to 255 characters in length. You must specify at least one SAN, and the list of SANs should include the common name.

```
$ request_body=$(< <(cat <<EOF
{
  "country": "US", 
  "state": "MA",
  "locality": "Boston",
  "organization": "City of Boston",
  "organizational_unit": "Championship Parade Detail",
  "common_name": "app.example.com",
  "sans": [
    "app.example.com",
    "www.example.com",
    "blog.example.com",
    "example.com"
  ],
  "key_type" : "p256v1"
}
EOF
))
```

### 2. Request a CSR that can be provided to your customer

```
$ curl -sXPOST https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_csrs\
    -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}"\
    -H 'Content-Type: application/json' -d "$request_body" 

{
  "result": {
    "id": "7b163417-1d2b-4c84-a38a-2fb7a0cd7752",
    "country": "US",
    "state": "MA",
    "locality": "Boston",
    "organization": "City of Boston",
    "organizational_unit": "Championship Parade Detail",
    "common_name": "app.example.com",
    "sans": [
      "app.example.com",
      "www.example.com",
      "blog.example.com",
      "example.com",
    ],
    "key_type": "p256v1",
    "csr": "-----BEGIN CERTIFICATE REQUEST-----\nMIIBSzCB8gIBADBiMQswaQYDVQQGEwJVUzELMAkGA1UECBMCTUExDzANBgNVBAcT\nBkJvc3RvbjEaMBgGA1UEChMRQ2l0eSBvZiBDaGFtcGlvbnMxGTAXBgNVBAMTEGNz\nci1wcm9kLnRscy5mdW4wWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAaTKf70NYlwr\n20P6P8xj8/4mTN5q28dbZR/gM3u4m/RPs24+PxAfMZCNvkVKAPVWYfUAadZI4Ha/\ndxLh5Q6X5bhIoC4wLAYJKoZIhvcNAQkOMR8wHTAbBqNVHREEFDASghBjc3ItcHJv\nZC50bHMuZnVuMAoGCCqGSM49BAMCA0gAMEUCIQDgtFUZav466SbT2FGBsIBlahDI\nVkg4y+u+V/K5DlY1+gIgQ9xLfUSKnSnJYbM9TwWr4Z964+lBtB9af4O5pp7/PSA=\n-----END CERTIFICATE REQUEST-----\n"
  },
  "success": true,
}
```

Note that the '\n' strings should be replaced with actual newline before passing to your customer. This can be accomplished by piping the output of the prior call to a tool like jq and perl, e.g.,:

```
$ curl -sXPOST https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_csrs\
    -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}"\
    -H 'Content-Type: application/json' -d "$request_body" | jq .result.csr |\
    perl -npe s’/\\n/\n/g; s/”//g’ > csr.txt
```

### 3. Pass the CSR to your customer and obtain a certificate from them to be uploaded.

Your customer will take the provided CSR and work with their CA to obtain a signed, publicly trusted certificate.

## Uploading the certificate

Upload the certificate and reference the ID that was provided when you generated the CSR.

You should replace newlines in the certificate with literal '\n' characters, as illustrated above in the custom certificate upload example. After doing so, build the request body, providing the ID that was returned in step #2.

Note that Cloudflare only accepts publicly trusted certificates; if you attempt to upload a self-signed certificate it will be rejected.

```
$ MYCERT="$(cat app_example_com.pem|perl -pe 's/\r?\n/\\n/'|sed -e 's/..$//')"
$ request_body=$(< <(cat <<EOF
{
  "hostname": "app.example.com",
  "ssl": {
    "custom_csr_id": "7b163417-1d2b-4c84-a38a-2fb7a0cd7752",
    "custom_certificate": "$MYCERT"
  }
}
EOF
))
```

With the request body built, create the Custom Hostname with the supplied custom certificate. Note that if you intend to use the certificate with multiple hostnames, you will need to make multiple API calls replacing the “hostname” field with each hostname you wish to create. 

```
$ curl -sXPOST https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_hostnames\
    -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}"\
    -H 'Content-Type: application/json'\
    -d "$request_body" 

{
  "result": {
    "id": "d03e8f82-8ad0-4939-a600-82c30261f808",
    "hostname": "app.example.com",
    "ssl": {
      "id": "48ab2d69-4b49-4ee3-88a2-0e6b2a3446b0",
      "status": "pending_deployment",
      "hosts": [
        "app.example.com",
        "www.example.com",
        "blog.example.com",
        "example.com"
      ],
      "issuer": "DigiCertInc",
      "status": "ECDSAWithSHA256",
      "uploaded_on": "2017-11-17T04:33:54.561747Z",
      "expires_on": "2018-11-21T12:00:00Z",
      "custom_csr_id": "7b163417-1d2b-4c84-a38a-2fb7a0cd7752",
    }
  },
  "success": true
}
```

## Listing and Deleting

### 1. List the custom CSRs that were previously generated in an account.

You can request the (paginated) collection of all previously generated custom CSRs by making a GET request to https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_csrs.

Here is an example in an account which has two CSRs for the same hostnames—one using an ECC P-256 key and the other using an RSA 2048-bit key:

```
{
    "result": [
      {
    "id": "7b163417-1d2b-4c84-a38a-2fb7a0cd7752",
    "country": "US",
    "state": "MA",
    "locality": "Boston",
    "organization": "City of Boston",
    "organizational_unit": "Championship Parade Detail",
    "common_name": "app.example.com",
    "sans": [
      "app.example.com",
      "www.example.com",
      "blog.example.com",
      "example.com",
    ],
    "key_type": "p256v1",
    "csr": "-----BEGIN CERTIFICATE REQUEST-----\nMIIB...PSA=\n-----END CERTIFICATE REQUEST-----\n"
    },
    {
    "id": "2b193417-ad2b-1c6d-339f-a6b7a0cd790a",
    "country": "US",
    "state": "MA",
    "locality": "Boston",
    "organization": "City of Boston",
    "organizational_unit": "Championship Parade Detail",
    "common_name": "app.example.com",
    "sans": [
      "app.example.com",
      "www.example.com",
      "blog.example.com",
      "example.com",
    ],
    "key_type": "rsa2048",
    "csr": "-----BEGIN CERTIFICATE REQUEST-----\nMIIS...FAE=\n-----END CERTIFICATE REQUEST-----\n"
    }
    ],
    "result_info": {
        "page": 1,
        "per_page": 20,
        "count": 2,
        "total_count": 2,
        "total_pages": 1
    },
    "success": true
}
```

2. Optionally, delete one or more of the CSRs to delete the underlying private key.

You may delete a CSR provided there are no custom certificates using the private key that was generated for the CSR. If you attempt to delete a CSR whose private key is still in use, you will receive an error.

```
$ curl -sXDELETE https://api.cloudflare.com/client/v4/zones/{zone_id}/custom_csrs/{csr_id}\
    -H "X-Auth-Email: {email}" -H "X-Auth-Key: {key}" 

{
    "result": null,
    "success": true,
    "message": "Custom CSR deleted successfully"
}
```