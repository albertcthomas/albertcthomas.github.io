+++
type = "post"
title = "Solving an SSL certificate error"
date = 2025-03-03
+++

This post is a follow-up to my previous one on [python development behind a proxy server](../python_development_behind_proxies), which I recommend reading first. In that post, I explain how to use `REQUESTS_CA_BUNDLE` to specify the location of the SSL certificate bundle for `pip` and `conda`. However, this applies specifically to the `requests` python library, which both `conda` and `pip` rely on internally—**but not everything in Python does**.

Recently, while working with [`autogluon`](https://auto.gluon.ai/dev/index.html), I wanted to try the multimodal configuration on a tabular dataset containing text columns. When running it, I encountered the following SSL error: 
```bash
[nltk_data] Error loading averaged_perceptron_tagger: <urlopen error
[nltk_data]     [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify
[nltk_data]     failed: self-signed certificate in certificate chain
[nltk_data]     (_ssl.c:1006)>
```
It appears that the underlying library detected my company's self-signed certificate but lacked the necessary certificate authority (CA) to validate it. Since my company's certificates are already included in my system-wide trusted CA list at `/etc/ssl/certs/ca-certificates.crt`, this meant only one thing: the underlying library wasn’t looking in the right place.

Digging deeper into the error message and `nltk`, I discovered that the request was made using `urllib` and I found that `urllib` relies on Python's `ssl` module for certificate validation. To check where `ssl` is looking for the certificate, I ran:
```python
import ssl
print(ssl.get_default_verify_paths())
```
As expected, the location used was not the system-wide trusted CA list. The environment variable to use for the `ssl` module is`SSL_CERT_DIR`. Setting `SSL_CERT_DIR` to `/etc/ssl/certs/ca-certificates.crt` fixed the issue[^1].

Unfortunately again, there doesn’t seem to be a single, universal environment variable that all applications agree on for locating SSL certificates. Instead, we have a mess of different environment variables (`REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE`, `SSL_CERT_DIR`, ...), and each tool decides for itself which one to follow.

[^1]: `/etc/ssl/certs/ca-certificates.crt` is not a directory but this worked for me. You can also use `SSL_CERT_FILE`.