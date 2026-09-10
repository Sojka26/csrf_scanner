# CSRF Candidate Scanner

A lightweight Python tool for identifying **potential Cross-Site Request Forgery (CSRF) weaknesses** in web applications.

The scanner passively crawls pages within the same origin and analyzes HTML forms, possible CSRF tokens, potentially state-changing GET requests, and selected cookie security attributes.

> **Important:** Use this tool only against applications you own or have explicit authorization to test.

## Features

The scanner can:

* crawl pages within the same origin;
* discover HTML forms and their input fields;
* identify POST forms without an obvious CSRF token;
* recognize common CSRF token names;
* flag potentially state-changing GET forms;
* inspect `Set-Cookie` headers for `SameSite`;
* detect apparent `SameSite=None` cookies without `Secure`;
* detect CSRF-related `<meta>` elements;
* limit the number of crawled pages;
* generate a text report containing discovered candidates.

The scanner identifies **candidates**, not confirmed vulnerabilities.

## Requirements

* Python 3
* `requests`
* `beautifulsoup4`

## Installation

Clone or download the project and install the required dependencies:

```bash
pip install requests beautifulsoup4
```

Using a virtual environment is recommended:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install requests beautifulsoup4
```

On Windows:

```powershell
.venv\Scripts\Activate.ps1
```

## Usage

Basic usage:

```bash
python3 csrf_scanner.py https://example.com
```

You can also specify the crawl limit, request timeout, and output filename:

```bash
python3 csrf_scanner.py https://example.com \
    --max-pages 100 \
    --timeout 15 \
    --output csrf_report.txt
```

## Command-Line Arguments

```text
url             Base URL to scan

--max-pages     Maximum number of pages to crawl
                Default: 50

--timeout       HTTP request timeout in seconds
                Default: 10

--output        Output report filename
                Default: csrf_report.txt
```

## How It Works

### 1. Same-Origin Crawling

The scanner starts from the supplied URL and follows discovered HTTP/HTTPS links.

Only URLs belonging to the same origin are added to the crawl queue.

For example, when scanning:

```text
https://example.com
```

a link to:

```text
https://example.com/account
```

can be crawled, while an external link is ignored.

### 2. CSRF Token Detection

The scanner looks for form fields whose names resemble common anti-CSRF token conventions.

Recognized patterns include:

```text
csrf
xsrf
_token
authenticity_token
requestverificationtoken
anti-forgery
nonce
```

For example:

```html
<input
    type="hidden"
    name="csrf_token"
    value="..."
>
```

will be recognized as a possible CSRF protection mechanism.

Token detection is based on the **field name only**. The scanner does not verify whether the token is cryptographically secure, session-bound, unpredictable, or actually validated by the server.

## POST Form Analysis

When a POST form does not contain an obvious CSRF token, the scanner creates a:

```text
CSRF candidate
```

Example:

```html
<form action="/account/email" method="POST">
    <input name="email">
    <button type="submit">Save</button>
</form>
```

Possible report entry:

```text
CSRF candidate
URL: https://example.com/account/email
Evidence: POST form without an obvious CSRF token. Fields: email
```

The absence of a visible token does **not** prove that the endpoint is vulnerable.

The application may use other protections, including:

* `Origin` validation;
* `Referer` validation;
* `SameSite` cookies;
* custom request headers;
* framework-level CSRF defenses;
* other server-side controls.

## State-Changing GET Candidates

The scanner also looks for GET forms that appear potentially state-changing.

It searches the form action and field names for words such as:

```text
delete
remove
update
edit
change
create
save
password
email
profile
account
transfer
payment
checkout
order
logout
```

For example:

```html
<form action="/account/delete" method="GET">
    <button type="submit">Delete</button>
</form>
```

may be reported as:

```text
State-changing GET candidate
```

This is a heuristic check and may produce false positives.

## Cookie Analysis

The scanner inspects observed `Set-Cookie` headers for selected CSRF-related security properties.

It can report:

```text
Cookie CSRF hardening candidate
```

when a `Set-Cookie` header is observed without an obvious `SameSite` attribute.

It can also report:

```text
Cookie configuration candidate
```

when it appears that `SameSite=None` is being used without `Secure`.

These findings should be manually verified because cookie handling can be more complex than a single response header indicates.

## CSRF Meta Tokens

The scanner checks `<meta>` elements for names matching its CSRF token patterns.

For example:

```html
<meta
    name="csrf-token"
    content="..."
>
```

may generate:

```text
CSRF token metadata
```

This indicates that a possible CSRF token mechanism exists on the page. It does not verify how the application uses that token.

## Output

After crawling finishes, the scanner creates a text report.

Example:

```text
CSRF CANDIDATE REPORT
================================================================================

Pages scanned: 15
Candidates: 2

[1] CSRF candidate
URL: https://example.com/account/email
Evidence: POST form without an obvious CSRF token. Fields: email
--------------------------------------------------------------------------------

[2] State-changing GET candidate
URL: https://example.com/account/delete
Evidence: GET form appears potentially state-changing. Fields: account
--------------------------------------------------------------------------------
```

## Finding Types

| Finding                           | Meaning                                                      |
| --------------------------------- | ------------------------------------------------------------ |
| `CSRF candidate`                  | POST form without an obvious CSRF token                      |
| `CSRF protected form candidate`   | POST form contains a field resembling a CSRF token           |
| `State-changing GET candidate`    | GET form appears potentially state-changing                  |
| `Cookie CSRF hardening candidate` | Observed cookie header does not appear to specify `SameSite` |
| `Cookie configuration candidate`  | Apparent `SameSite=None` configuration without `Secure`      |
| `CSRF token metadata`             | CSRF-related `<meta>` element was discovered                 |

## Limitations

This is a **candidate scanner**, not an exploit or vulnerability verification tool.

The scanner does not:

* submit discovered POST forms;
* perform state-changing CSRF attacks;
* verify whether an anti-CSRF token is actually validated;
* determine whether a token is unpredictable;
* test token reuse;
* automatically authenticate to applications;
* execute JavaScript in a browser;
* fully analyze dynamically generated SPA forms;
* prove that a missing token results in exploitable CSRF;
* determine the business impact of a discovered endpoint.

False positives are therefore expected.

## Interpreting Results

A report such as:

```text
POST form without an obvious CSRF token
```

should be treated as a starting point for manual investigation rather than proof of a vulnerability.

A meaningful CSRF vulnerability generally requires a security-relevant action that can be triggered in the victim's authenticated context without adequate protection.

For example, a missing token on a public search form may have little or no security impact, while the same condition on an authenticated account-change operation deserves significantly more attention.

## Safe Testing

Only test systems for which you have authorization.

When manually validating a candidate:

1. Use a dedicated test account.
2. Record the account state before testing.
3. Avoid production data and real transactions.
4. Determine whether the endpoint actually changes server-side state.
5. Check whether CSRF protection exists outside the form itself.
6. Use harmless, reversible changes where possible.
7. Restore the test account to its original state afterward.
8. Document the request, response, observed state change, and relevant protection mechanisms.

Do not classify a finding as confirmed CSRF solely because a form lacks a hidden token.

## Example

Run a small scan:

```bash
python3 csrf_scanner.py https://example.com \
    --max-pages 20 \
    --output csrf_report.txt
```

The generated report can then be reviewed manually to determine which candidates represent security-sensitive functionality.

## Responsible Use

This project is intended for:

* authorized penetration testing;
* security research in controlled environments;
* application security reviews;
* defensive security testing;
* educational labs.

Do not use it to scan systems without permission.

## Disclaimer

The tool provides heuristic security findings and may produce both false positives and false negatives.

Users are responsible for ensuring that testing is authorized and performed safely.

## License

Add the license appropriate for your project, such as the MIT License.
