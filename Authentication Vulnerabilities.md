# PortSwigger Web Security Academy — Authentication Vulnerabilities

## Quick Reference / Notes

- 3 types of authentication:

  * **Something you know**
  * **Something you have**
  * **Something you are**
- Authentication mechanisms can be vulnerable because of:

  * Weak authentication mechanisms that do not properly protect against brute-force attacks.
  * Logic flaws or poor implementation. This is commonly referred to as **Broken Authentication**.
- Brute-forcing usernames:

  * Business logins are often predictable, e.g. `firstname.lastname@somecompany.com`
  * Other common usernames include `admin` and `administrator`.
- When brute-forcing a login page, pay attention to:

  * HTTP status codes
  * Error messages
  * Response times

---

## Lab: Username enumeration via different responses

1. Sent the `POST` login request to **Burp Intruder**. It contained username and password parameters.
2. Set the username parameter as the payload position.
3. Used the username list provided by Burp Suite.
4. Ran a **Sniper** attack with a simple list.
5. One response had a slightly longer length than the others.
6. This response indicated that the username was valid but the password was incorrect.
7. Set the discovered username and moved the payload position to the password.
8. Used the provided password list to brute-force the password.
9. The response with the different length indicated the correct password.

## Lab: Username enumeration via subtly different responses

1. Sent the `POST` login request to **Burp Intruder**.
2. Set the username as the payload position and selected **Simple list**.
3. Added the username list to the payload configuration.
4. In **Settings**, used **Grep - Extract** to extract the error message.
5. Started the attack and compared the responses.
6. The difference was subtle — one response had a `.` while another had a trailing space.
7. Used the username identified from this difference.
8. Moved the payload position to the password and used the password list.
9. Kept the same grep setting.
10. This time, the correct response had a **302** status code instead of **200**.

## Lab: Username enumeration via response timing

1. Started by looking at the login request.
2. Since the previous method could not be used, I tried **IP spoofing** using the `X-Forwarded-For` header.
3. Set up the header in Burp:

   * **Proxy → Settings**
   * Set the header to `X-Forwarded-For`
4. Sent the request to **Intruder**.
5. Set two payload positions:

   * `X-Forwarded-For`
   * Username
6. For the `X-Forwarded-For` payload, used a number list from `1` to `100`.
7. For the username payload, used the provided username list.
8. Set the password to almost 100 characters.
9. Started the attack and compared response times.
10. The username with the highest response time was the valid username.
11. Set the correct username and changed the second payload to the password list.
12. The response with a **302** status code indicated the correct password.

---

## Flawed Brute-Force Protection

Common ways to prevent brute-force attacks:

1. Lock the account after too many failed login attempts.
2. Block the remote user's IP address after too many login attempts in succession.
3. Include your own valid login credentials at regular intervals throughout the wordlist.

---

## Lab: Broken brute-force protection, IP block

1. Sent the login request to **Repeater**.
2. After 3 continuous failed login attempts, the account was blocked.
3. However, logging in with my own account before reaching the limit reset the failed-attempt counter.
4. Sent the request to **Intruder**.
5. Set payloads for both username and password.
6. The username list alternated between my own username and the victim username.
7. Added my own valid password between each password in the password list.
8. This meant that during the attack, my valid username/password combination would periodically reset the failed-attempt counter.
9. Started the attack and looked for the pattern where the valid login responses appeared.
10. The remaining valid-looking response identified the victim's credentials.

---

## Account Locking

Ways to work around account locking:

1. Establish a list of candidate usernames that are likely to be valid.
2. Create a shortlist of passwords that at least one username is likely to have.

   * The number of passwords should not exceed the allowed number of login attempts.
   * For example, if the limit is 3 attempts, use a maximum of 3 password guesses.
3. Use Burp Intruder to try each selected password against each candidate username.

- Account locking does not protect against **credential stuffing**.
- Credential stuffing uses large lists of username/password pairs from previously compromised accounts.
- It relies on users reusing the same username and password across multiple websites.

---

## Lab: Username enumeration via account lock

1. Sent the login request to **Intruder**.
2. Set a payload position for the username and another blank payload position after the password.
3. The request looked like:

```text
username=$user$&password=pwd$$
```

4. Selected **Cluster bomb**.
5. Used the username list for the first payload.
6. For the second payload, selected a **Null payload** and configured it to generate 5 payloads.
7. This caused each username to be repeated 5 times.
8. Started the attack and compared the responses.
9. One response was longer and contained the **too many login attempts** message.
10. This identified a valid username.
11. Started another attack using **Sniper**.
12. Set the discovered username and moved the payload position to the password.
13. Used the password list and kept the grep extraction for the error message.
14. One response differed from the others and indicated the correct password.

---

## User Rate Limiting

- User rate limiting blocks too many login attempts from the same IP within a short period.
- An IP may be unblocked:

  1. Automatically after a certain period.
  2. Manually by an administrator.
  3. Manually by the user after successfully completing a CAPTCHA.
- If the limit is based on the number of HTTP requests from an IP address, it may be possible to bypass it by guessing multiple passwords within a single request.

---

## Lab: Broken brute-force protection, multiple credentials per request

1. Sent the login request to **Repeater**.
2. Observed that the parameters were in JSON format.
3. This meant it was possible to test whether the server accepted an array of passwords in a single request.
4. Pasted an array of passwords with the target username.
5. The response returned a **302** status code.
6. Opened the response in the browser.
7. The lab was solved.

### Key Takeaway

If a login endpoint accepts JSON and does not properly validate that the password is a single string, it may be possible to send multiple passwords in one request. A flawed implementation could then iterate through the array and bypass simple request-based rate limiting.

---

## HTTP Basic Authentication

- The client receives an authentication token from the server.
- The token is the username and password concatenated together and encoded using Base64.
- The browser stores and manages this token and automatically sends it in the `Authorization` header with subsequent requests.
- Example:

```text
Authorization: Basic base64(username:password)
```

- HTTP Basic Authentication has several security concerns:

  * Credentials are repeatedly sent with requests.
  * Without HSTS, there is potential for interception through a man-in-the-middle attack.
  * It does not provide strong protection against brute-force attacks.
  * It can be vulnerable to session-related attacks such as CSRF.

---

## Two-Factor Authentication Tokens

- A second authentication factor can also be abused.
- Examples include:

  * Verification codes sent through SMS, which can introduce interception risks.
  * **SIM swapping**, where an attacker obtains control of the victim's SIM.
- Potential 2FA bypasses:

  1. Check whether the user is considered logged in before entering the verification code.
  2. After completing the first authentication step, try accessing pages that should only be available after the second step.
  3. Check whether the website actually verifies that the second authentication step was completed before loading protected pages.

---

## Lab: 2FA simple bypass

1. Logged in using the provided account to understand how the 2FA process worked.
2. Checked the URLs and requests involved in the process.
3. Logged out.
4. Logged in using the target account's credentials for the first step.
5. Used the email functionality and observed the resulting page.
6. Returned to the home page.
7. Clicked **My account**.
8. The target user's account page opened without requiring the second authentication step.

---

## Flawed Two-Factor Verification Logic

- A flawed 2FA implementation may not properly verify that the same user completed both authentication steps.
- For example:

  1. User logs in with their normal credentials in the first step:

```http
POST /login-steps/first HTTP/1.1
username=carlos&password=qwerty
```

2. The server assigns a cookie related to the account before sending the user to the second step:

```http
Set-Cookie: account=carlos
```

3. The second-step request uses this cookie to identify the account:

```http
GET /login-steps/second HTTP/1.1
Cookie: account=carlos
```

4. If the server trusts this cookie without properly validating the authentication flow, an attacker could potentially change it to another username.
5. If the verification code can then be brute-forced, the attacker could potentially access another user's account without knowing their password.

---

## Lab: 2FA broken logic

1. Logged in with the provided account to understand the authentication flow.
2. Sent the `POST /login2` request to **Repeater**.
3. Changed the username to the target user and sent the request.
4. The response showed an invalid MFA code.
5. Brute-forced the MFA code.
6. Identified the response with the successful status code.
7. Sent the successful request through Repeater.
8. Opened the response in the browser to complete the lab.

---

## Brute-Forcing 2FA Verification Codes

- Some websites prevent brute-forcing by automatically logging the user out after a certain number of incorrect verification codes.
- Burp macros can be used to automate the multiple steps required for each login attempt.
- **Burp Intruder** or **Turbo Intruder** can then be used to automate the verification-code requests.

---

## Lab: 2FA bypass using a brute-force attack

1. Checked the `login` and `login2` process and inspected the requests in Burp Suite.
2. Went to:

   * **Settings → Session handling → Add rule**
3. In the **Scope** tab, included all URLs.
4. In the **Details** tab, added a rule action to **Run a macro**.
5. Selected the sequence:

   * `GET login`
   * `POST login`
   * `GET login2`
6. Tested the macro and checked that the final response contained the verification-code page.
7. Sent the `login2` request to **Intruder**.
8. Set the payload to the MFA code:

   * Number list
   * `0` to `9999`
   * Step: `1`
   * Minimum digits: `4`
   * Maximum digits: `4`
9. Set the resource pool to a maximum of **1 concurrent request**.
10. Started the attack.
11. The response with the successful status code identified the correct MFA code.
12. Had to run the attack multiple times because the code expired after some time.

---

## Stay-Logged-In Cookies

- Other authentication mechanisms can also contain vulnerabilities, including password reset and **Remember me / Stay logged in** functionality.
- Stay-logged-in functionality often uses a persistent cookie.
- If an attacker obtains this cookie, they may be able to bypass the normal login process.
- A cookie should be sufficiently unpredictable to prevent guessing.
- Some vulnerable implementations construct cookies using predictable information such as:

  * Username
  * Timestamp
  * Password
- If an attacker can create their own account, they can inspect their own cookie and try to determine how it was generated.
- Base64 encoding or hashing does not automatically make a cookie secure.
- If a predictable hashing method is used without a salt, an attacker may be able to brute-force the value.

---

## Lab: Brute-forcing a stay-logged-in cookie

1. Logged in using the provided account and sent the request containing the stay-logged-in cookie to **Repeater**.
2. Decoded the cookie and identified the username within it.
3. Used Hackvector to test how the password portion was generated.
4. Identified that the password was hashed and the resulting value was then Base64 encoded together with the username.
5. Sent the request containing the ID and stay-logged-in cookie to **Intruder**.
6. Set the stay-logged-in cookie as the payload position.
7. Used the password list as the payload.
8. Added rules to:

   * MD5 hash the password
   * Add the username as a prefix
   * Base64 encode the final value
9. Used **Grep - Match** to identify the response containing the account functionality that is only available after successful authentication.
10. Ran the attack and identified the response that matched.
11. Opened the successful response in the browser.

## Lab: Offline password cracking

1. Looked around the website and checked the blog and comment section.
2. Tested whether the comment section was vulnerable to XSS by submitting:

```html
<script>alert("hi!")</script>
```

3. The script executed on the home page, confirming that the comment was being rendered as HTML/JavaScript.
4. Checked the exploit server and its access log.
5. Used the comment functionality to obtain the relevant authentication information through the exploit server.
6. Found a secret key and a stay-logged-in cookie in the access log.
7. Sent the `/my-account` request to **Repeater**.
8. Changed the account ID to the target user and replaced the stay-logged-in cookie with the value obtained from the access log.
9. The successful response showed that the cookie was valid.
10. Decoded the cookie using Base64.
11. The password portion was an MD5 hash.
12. Used the known structure of the cookie to identify the password.
13. Logged in with the recovered credentials and completed the required account action.

---

## Password Reset

### Resetting Passwords via Email

- Sending passwords to users by email is generally not a good approach.
- Some websites generate a new password and send it through email.
- Persistent passwords should not be sent over an insecure channel.
- If a password is sent through email, security depends on factors such as:

  * The password expiring very quickly.
  * The user changing the password immediately after receiving it.

### Resetting Passwords Using URLs

- A more robust method is to send the user a unique URL that takes them to the password reset page.
- A weaker implementation may use an easy-to-guess parameter identifying the account:

```text
http://vulnerable-website.com/reset-password?user=victim-user
```

- If the application trusts this parameter, an attacker may be able to change it to another user's username and reset their password.
- A better implementation generates a **high-entropy, hard-to-guess token**.
- Ideally, the reset URL should not reveal which user's password is being reset.
- Example:

```text
http://vulnerable-website.com/reset-password?token=<high-entropy-token>
```

- When the user visits the URL, the server should:

  1. Check that the token exists.
  2. Identify the associated account.
  3. Ensure the token has not expired.
  4. Destroy the token after the password has been reset.
- A potential flaw occurs when the application does not validate the token again when the reset form is submitted.
- An attacker could potentially access a reset page from their own account, remove or invalidate the original token, and then use the same page to reset another user's password.

---

## Lab: Password Reset Broken Logic

1. Followed the normal password-reset process for wiener:peter: clicked Forgot password, entered the username, checked the email inbox for the reset link, and set a new password.
2. Checked the HTTP history and inspected the POST request sent when setting the new password.
3. Found the following parameters:
```text
temp-forgot-password-token=t1ucg90pj0zkbaqe6zzp8xsolbj53xne&username=wiener&new-password-1=peter1&new-password-2=peter1
```
4. Sent the request to Repeater.
5. Based on the theory, I understood that the token was not being validated again.
6. Removed the entire token parameter from the URL and request body and changed the username to carlos. The request looked like this:
```text
POST /forgot-password HTTP/2
username=carlos&new-password-1=peter1&new-password-2=peter1
```
7. However, I only needed to remove the token's value, not the entire parameter.
8. The final request was:
```bash
POST /forgot-password?temp-forgot-password-token= HTTP/2
temp-forgot-password-token=&username=carlos&new-password-1=peter1&new-password-2=peter1
```
9. The response status was 302, which meant the password had been changed successfully.
10. Logged in to Carlos's account using the new password.
11. Opened My account and confirmed that the lab was solved.

---

## Password Reset Poisoning

- If the URL in a password-reset email is generated dynamically, an attacker may be able to manipulate it.
- The attacker can potentially steal another user's token and use it to change that user's password.

---

## Lab: Password Reset Poisoning via Middleware

1. Turned Intercept on in Burp Suite Proxy.
2. Went to My account → Forgot password and entered wiener.
3. Received the reset link in the email client on the exploit server, visited it, and changed the password to peter1.
4. Turned Intercept off. Checked the HTTP history and found the POST /forgot-password request that sent only the username. There was no token or other check in this request.
5. The follow-up POST request contained temp-forgot-password-token, new-password-1, and new-password-2 in the request body. Sent both requests to Repeater.
6. In the request that contained only the username, changed the username to carlos and sent the request.
7. Checked the exploit server's access log.
   <img height="100" alt="image" src="https://github.com/user-attachments/assets/88a61fc1-3a1a-4892-9fe7-e22262dfe731" />
8. Found Carlos's token, which I should not have been able to access.
9. Copied the token and replaced Wiener's token with Carlos's token in the password-reset request. Sent the request.
10. The response returned a 300 status, indicating that the password had been changed.
11. Went to My account in the browser and logged in as carlos:peter1.
12. Confirmed that the lab was solved.

---

## Changing User Passwords

- A typical password-change process requires the user to enter their current password and then their new password.
- This can be dangerous if the application allows an attacker to access the functionality directly without being logged in as the victim.
- For example, if the username is stored in a hidden field, an attacker might be able to modify it in the request and target arbitrary users.
- This can be exploited to enumerate usernames and brute-force passwords.

---

## Lab: Password Brute-Force via Password Change

1. Went through the password-change process for wiener:peter and set the new password to peter1. Captured all the requests in Burp Suite.
2. Inspected the POST requests and identified POST /my-account/change-password HTTP/2 as the useful request. Sent it to Repeater.
3. Set the two new-password parameters to different values.
   <img height="100" alt="image" src="https://github.com/user-attachments/assets/83fe845a-1898-4b04-a442-586ea61c83cb" />
4. The response showed the error New passwords do not match. It did not mention that the current password was wrong, which suggested that the current password was probably correct.
5. Sent the request to Intruder and configured the following:
   <img height="100" alt="image" src="https://github.com/user-attachments/assets/4eadb674-53d4-4c0a-9cd5-6a56791f508e" />
6. Used a Sniper attack to brute-force the password. While analysing the responses, I found one with a length of 4114 bytes, while the others were 4117 bytes.
7. Logged in to Carlos's account using the password dragon. The lab was solved.

---

## Securing Authentication Mechanisms

- Take care when handling user credentials.
- Do not rely on users to maintain security.
- Prevent username enumeration.
- Implement robust brute-force protection, such as strict IP-based user rate limiting and CAPTCHA tests.
- Triple-check verification logic. A check that can be bypassed is no better than having no check at all.
- Do not forget supplementary functionality.
- Implement proper multi-factor authentication (MFA).


