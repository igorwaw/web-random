Title: Reading Outlook emails with Python and O365
Date: 2024-03-10 19:00
Status: draft
Category: random
Tags: python, microsoft, authentication
Slug: outlook-emails-python

Recently I needed to write a script that checks for new email and does something based on the contents. I tested it on a Gmail account with IMAP enabled, in a few hours I got a working code with some bells and whistles (retries with configurable delay, proper exception handling, built-in support for exporting Prometheus metrics).

But the company hosts their email on Microsoft servers. Easy, I'll just change the server hostname, right? Wrong! Microsoft disabled basic authentication (with username and password) for IMAP two years ago, I need to use OAauth2. OK, I used OAuth2 before, it's sometimes a pain but shouldn't take more than an hour, right? Wrong again! Microsoft's OAuth2 and Graph API turned out to be a nightmare.

It's not that it's not documented. On the contrary - there's a long documentation detailing 3 different authentication flows and countless permissions. It's probably good for security that it allows a very fine grained access control. But I just want to read emails without getting a PhD in Authentication Science.

There are also some forum posts, Reddit, StackOverflow, usual places. Since there are different possible flows and also Microsoft deprecated some APIs, it's hard to know which posts are relevant.

## MSAL didn't make it

Microsoft provides a library called MSAL. My script would first authenticate using MSAL to get a token. That sort of worked. Then it would use this token to authenticate to the IMAP server. And that didn't.

Now, it's probably possible to do it, I just made a mistake somewhere. The problem is, there are many places where something can be set wrong. Application permissions, credentials, account settings, getting a wrong type of token, malformed request to IMAP server and hundreds more things. But on the client side the only information I got was "Authentication failed" and I couldn't get any logs from the server.

## O365 library to the rescue

I found another library O365 which aims to make interacting with Graph API and Office 365 easier. And it's not provided by Microsoft, which explains why the documentation is shorter and to the point.

First step is to install the library - simple `pip install O365` worked fine, both on my laptop where I developed it and on the Alpine-based container I used in production.

Now the app registration. There are some non-intuitive things:

- Go to the [Azure portal](https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade), select App registrations and New registration. Invent a name.
- For the app type choose "Desktop or mobile". My script is neither, but it's the closest thing - it's not a web app that can expose a callback URL.
- But we still need to provide the URL! Use this one: `https://login.microsoftonline.com/common/oauth2/nativeclient`
- Go to "Certificates and secrets" and generate a new application secret. Copy it somewhere, it will be hidden later.
- You'll also need application (client) ID and tenant ID, you can find them on the Overview.
- Go to API permissions, click "Add a permission" and "Microsoft Graph".

Here's where the path diverges. `Delegated permissions` require a user consent to connect. `Application permissions` don't require it and get permission to the whole organization, so they require admin approval. I suggest to use the first one, at least for the experiments, despite the inconvenience.

- Choose `Mail.ReadWrite` and `offline_access`. Mail.Read would of course be enough to read emails, but wouldn't allow to mark message as read, and offline_access is needed for automatic token renewal.

## Authenticating with delegated permissions

Time to write some code. For the test I hardcoded the credentials, the real containerized app will read them from the environment variables.

```python
from O365 import Account


def mslogin():
 
    mailtenantid = "tenant id here"
    mailsecret = "app secret here"
    mailclientid = "app client id here"

    print("Connecting to O365")
    scopes = ["Mail.ReadWrite", "offline_access"]
    account = Account(
        credentials=(mailclientid, mailsecret),
        tenant_id=mailtenantid,
        auth_flow_type="authorization",
    )
    if not account.is_authenticated:
        if account.authenticate(scopes=scopes):
            print("Authenticated!")
        else:
            print("Authentication failure")
    else:
        print("Token still valid, no need to authenticate again")
    return account
```

The script will display a URL. You need to open it in a browser, login to your Microsoft account if you're not already logged in and approve access for the app. The browser will display another URL, which you now need to paste in the terminal.

O365 library will then store the authentication token. Next time you run the script, `account.is_authenticated` will return True and you won't need to approve again. The token is valid for an hour, but it's automatically renewed each time you connect. My script will check email every few minutes, so it can stay approved indefinetely.

## Reading new email

That was mostly shown in the documentation as an example. The only slightly tricky part was constructing the query to check unread messages.

```python
    account = mslogin()
    mailbox = account.mailbox()
    inbox = mailbox.inbox_folder()
    query = mailbox.new_query().on_attribute("isRead").equals(False)
    for message in inbox.get_messages(query=query):
        message.mark_as_read()
        print(message.subject)
        print(message.body[:50])
```
