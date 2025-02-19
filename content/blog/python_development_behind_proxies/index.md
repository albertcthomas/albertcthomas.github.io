+++
type = "post"
title = "Python development behind corporate proxies"
date = 2024-10-21
tags = ["python"]
+++

I have been working as a machine learning researcher in the industry for almost 10 years and essentially two big companies. I had to deal with proxy issues as part of my daily work, including installing Python packages with `conda` and `pip`, cloning `git` repositories, and accessing other web-based resources. Proxy issues can be frustrating and challenging, especially as they often involve code and concepts outside your usual expertise. In this post, I will share some insights that may help you understand how proxies work and reduce your apprehension about dealing with proxies issues. The goal here isn’t to provide an exhaustive guide to proxy configurations for every tool you might use, there are plenty of resources for that, starting with each tool's documentation.

As a machine learning researcher relying heavily on Python, the examples I will share here are based on my experiences with tools such as `conda` and `pip` on Linux systems. However, most of the concepts still hold in general and can help you debug or understand proxy issues with other tools.

## What is a proxy server? 
A proxy server acts as an intermediary between you (on the company network) and the internet. All the traffic between you and the internet passes through this proxy server. This allows companies to control what goes in or out, such as blocking access to specific websites or preventing the upload of large data to external storage services. For your applications to access the web and communicate with external services, they must be configured to route their traffic through the proxy server. Most of the time, especially in corporate environments, the proxy server requires authentication: a username and password.

![proxy_server](proxy_server_white.jpg "Proxy server")

In most cases, the proxy is configured by your company's IT team on your machine and your favorite web browser (e.g. Chrome, Firefox) automatically retrieves this configuration. Therefore, you usually do not encounter proxy issues when browsing the web on this machine. However, when working on other machines or using applications that don’t automatically detect the proxy, you must manually configure the proxy settings.

Keep in mind that while usually all external traffic requires the proxy, internal traffic does not. If you set up a proxy configuration without specifying exceptions for internal resources, those requests will fail.

## Proxy configuration
To configure an application to use a proxy server and have a successful connection, you need the proxy URL (e.g., `http://proxy_url/`), the port to use (typically appended to the URL, e.g., `http://proxy_url:proxy_port/`) and often authentication information, a username and a password, usually but not always, passed in the URL (e.g., `http://user_id:user_pwd@proxy_url:proxy_port/`). The authentication information allows the proxy server to identify the user and enforce appropriate permissions. If your credentials are incorrectly encoded or if the application fails to interpret them, you’ll likely encounter an HTTP 407 Authentication Error.

Although some applications require independent proxy configuration, most will check the following environment variables (on Linux systems). You can verify in your application’s documentation how to configure the proxy.
```bash
export http_proxy="http://user_id:user_pwd@proxy_url:proxy_port/"
export https_proxy="http://user_id:user_pwd@proxy_url:proxy_port/"
```
When using these environment variables you need to encode the special characters[^1]. Sometimes an `https` URL should be assigned to `https_proxy` (check the [urllib3 documentation](https://urllib3.readthedocs.io/en/stable/advanced-usage.html#http-and-https-proxies) for more details).

You can also define which domain should bypass the proxy using the `no_proxy` environment variable. This is useful when you want to access internal resources without disabling your proxy configuration.
```bash
export no_proxy = '.corporate_domain.com'
```
Using these environment variables will impact all applications that rely on them. If you need different configurations for different applications, consider using the specific configuration options provided by each application.

In some cases, this setup is sufficient and you will be fine. The main challenge is ensuring that your application reads and interprets the configuration correctly. The next potential obstacle, often more confusing for users, is SSL errors.

## SSL/TLS verification
When communicating with an external website using HTTPS, SSL/TLS certificates are used to ensure that the website you're communicating with is genuinely the one you intend to reach, and not an imposter. Your system maintains a trusted list of Certificate Authorities (CAs), and when you establish a connection, it checks that the website's certificate has been issued by one of these trusted CAs. This process happens automatically on most operating systems. However **in a corporate environment, the corporate proxy might replace the original website's certificate with one it has issued itself, and therefore the proxy acts as a Certificate Authority**. If the proxy's certificate isn't included in your system's list of trusted CAs, you'll encounter an SSL/TLS error.

Disabling SSL verification is an option but is strongly discouraged, because it prevents you from verifying the authenticity of the connection. This is also why some applications make it very difficult to disable SSL verification. The preferred solution is to obtain your corporate proxy's certificate and add it to your system's trusted list or another list of certificates that you want your application to use[^2].

You can obtain your corporate certificate by asking your IT administrators or through your [web browser](https://smarie.github.io/develop-behind-proxy/know_your_proxy/#download-your-proxys-root-certificate). Often, if the company has pre-configured your machine, the proxy settings and certificates are already in place, so browsing usually works without issues and this is why you can get the certificate from your browser.

On Ubuntu systems the system-wide trusted CA list is located at `/etc/ssl/certs/ca-certificates.crt`. To add a new certificate to this list:
1. Place the `.crt` certificate file in `/usr/local/share/ca-certificates/` which is a directory reserved for custom CA certificates that users want to trust beyond the default ones.
2. Run `sudo update-ca-certificates` to add it to the system-wide trusted CA store. You might need to restart the service/tool you want to use to make sure it takes the updated list of certificates into account.

Some applications may not use the system-wide trusted CA list by default. For instance, the Python `requests` package might rely on the `certifi` Python package which provides a carefully curated list of Certificate Authorities. Running `python -m certifi` will get you the location of the file concatenating all the certificates if you want to use it instead of the system-wide CA list.

## Conda and Pip
It’s helpful to understand how `conda` and `pip` handle external requests and how to configure them to work behind a proxy.

To handle their HTTP requests, including package downloads, both `conda` and `pip` use the [`requests` Python HTTP library](https://requests.readthedocs.io/en/latest/), which itself relies on the [`urllib3` HTTP client for Python](https://urllib3.readthedocs.io/en/stable/). The `requests` library is designed to be more user-friendly and higher-level than `urllib3`[^3][^4].

The `requests` library can read the proxy configuration from environment variables, so this is how I configure the proxy for `pip` and `conda` on my Ubuntu machines[^5]:
1. I set the `http_proxy` and `https_proxy` environment variables to pass the proxy URL and authentication information, ensuring that I properly encode any special characters.
2. For SSL certificates, I place the corporate certificates in `/usr/local/share/ca-certificates/` and run `sudo update-ca-certificate` after putting my custom certificates in `/usr/local/share/ca-certificates/`. I then set the `REQUESTS_CA_BUNDLE` environment variable to specify the CA list location used by `requests`: 
```
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
```

Be aware that the default CA location used by `requests` might not be the system-wide CA list. For instance, it could the path returned by `python -m certifi`. To check the default location, run:
```python
from requests.utils import DEFAULT_CA_BUNDLE_PATH
print(DEFAULT_CA_BUNDLE_PATH)
```

Make sure that the `http_proxy`, `https_proxy` and `REQUESTS_CA_BUNDLE` environment variables are set in the terminal, `tmux`/`screen` or Python session you are using. Finally, if you don't have sudo rights you can always get the CA list provided by `certifi`, append your corporate certificate to it and then set the `REQUESTS_CA_BUNDLE` environment variable accordingly.

## Testing and debugging
Diagnosing proxy errors can be challenging. I like to use the following `urllib3` code snippet as a sanity check when testing or debugging my proxy configuration. This approach has limitations as this directly uses `urllib3` which might not be the library handling the HTTP requests for the specific application you are trying to use. This also avoids many steps (reading the environment variables, encoding the special characters, making sure I am passing the proxy configuration to the application) but it can still help understand issues quickly and iterate faster.
```python
from urllib3 import ProxyManager, make_headers

# authentication information are not passed with the proxy url but
# with a header. do not encode special characters here
default_headers = make_headers(proxy_basic_auth="user_id:user_pwd")

proxy = ProxyManager(
    "http://proxy_url:proxy_port/",
    proxy_headers=default_headers,
)

# the url needs to be passed as "http(s)://..."
response = proxy.request("GET", "the_url_you_want_to_test")

if response.status == 200:
	print("Success")
```
Since `urllib3` does not read your environment variables, you must manually provide the proxy URL and authentication information. Do not encode your special characters when directly passing your password to `urllib3`.

If this test succeeds, it confirms that the connection can work for the specified URL, authentication information and default certificates, indicating that the configuration works. If this fails I try to make it pass before returning to the original application.

If you have an SSL error, you can try disabling SSL verification temporarily by passing `cert_reqs="CERT_NONE"` to `ProxyManager`:
```python
proxy = ProxyManager(
    "http://proxy_url:proxy_port/",
    proxy_headers=default_headers,
	cert_reqs="CERT_NONE",
)
```

Next, try passing the required certificates, using the `ca_certs` argument to provide the path to the certificate file:
```python
from pathlib import Path
ca_certs = Path("~") / "your_ca_certs.pem"

proxy = ProxyManager(
    "http://proxy_url:proxy_port/",
    proxy_headers=default_headers,
    ca_certs=ca_certs,
)
```


One important thing to be aware of is that **some proxies cache your configuration and credentials** making debugging more complex. For instance, if you first try the correct configuration and then try a wrong one, the connection can still be successful due to caching. Debugging and understanding proxy errors is annoying and not always easy. Debugging and understanding proxy errors with caching is even worse.

I hope this blog post will help you resolve your proxy issues or at least make you feel more confident when dealing with them. Even with experience, I sometimes encounter proxy or SSL/TLS errors that I can’t immediately explain, and debugging can be challenging, especially when the problem occurs deep within the connection chain.

## Remarks and resources
* It is a bit annoying that there are applications that will not read your `http_proxy` environment variables or will not support having authentication information. It would also have been nice to have an environment variable like `http_proxy` to pass the trusted CA list and that would be read by almost all applications as `REQUESTS_CA_BUNDLE` is an environment variable specific to `requests`.
* A website on proxies with a great guide for SSL/TLS certificates <https://smarie.github.io/develop-behind-proxy/know_your_proxy/>.
* There are tools such as CNTLM that encrypt your password, centralize the proxy configuration and work for applications that do not allow passing a username and password to the proxy configuration.
* Discussion on `HTTP_PROXY` or `http_proxy` and the different ways to pass multiple domains to `no_proxy`: <https://about.gitlab.com/blog/2021/01/27/we-need-to-talk-no-proxy/>

[^1]: See the “Reserved characters after percent-encoding” table on the [Percent-encoding Wikipedia page](https://en.wikipedia.org/wiki/Percent-encoding) for guidance.
[^2]: Ensure this list also includes other CAs, as the proxy might not always replace the original certificate.
[^3]: `pip` vendors `requests` and `urllib3`, meaning they are packaged directly within `pip` itself.
[^4]: Don't confuse the third party package `urllib3` and the `urllib` module available in the Python standard library.
[^5]: [`mamba`](https://mamba.readthedocs.io/en/latest/index.html), which serves as a faster alternative to `conda`, does not use the `requests` library. However, it still reads the same environment variables as `conda`, allowing the proxy to be configured in the same manner.