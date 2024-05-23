getmail-container
======================

***getmail-container*** is a docker image containing an IMAP server and email fetcher: [*dovecot*](http://en.wikipedia.org/wiki/Dovecot_(software)) and [*getmail6*](https://getmail6.org/). It is used for gathering emails from multiple accounts on a private IMAP server while using a remote SMTP server for sending emails.

This *docker* container is inspired the work of Joel Porquet:  
<http://web.archive.org/web/20200807233801/https://joel.porquet.org/wiki/hacking/getmail_dovecot/>

```
+-----------+              +-----------+               +--------------+
| ISP       |              | DOCKER    |               | LAPTOP       |
|           |              |           |           +-->|--------------|
| +-------+ | push/delete  | +-------+ | push/sync |   |  MAIL CLIENT +---+
| | IMAPS +----------------->| IMAPS +<------------+   +--------------+   |
| +-------+ |              | +-------+ |           |   +--------------+   |
| +-------+ |              |           |           |   | ANDROID      |   |
| | SMTP  |<-------+       |           |           +-->|--------------|   |
| +-------+ |      |       |           |               |  MAIL CLIENT +---+
+-----------+      |       +-----------+               +--------------+   |
                   +------------------------------------------------------+
Joel's original diagram.
```

Usage
=====

Required volumes:

- `/home`: mounted users directories (`Maildir` in fs layout, `sieve`, `.getmail`)
- `/etc/ssl/private`: mounted TLS certificates (`tls.crt`, `tls.key`)

Environment Variables
- `CRON`: getmail will be called by cronjob to retrieve emails. Specify your schedule in cron format. Default is every 30 minutes `CRON="*/30 * * * *"` (don't forget the quotes). Keep in mind that some mail providers will block high frequent retrieving.
- `TZ`= set local timezone. Default is `TZ=UTC`

Prepare your getmailrc account configurations per user (`/srv/mail/home/user/.getmail/getmailrc-user@email.invalid`):

```
# ~/.getmail/getmailrc-*: getmailrc email configuration

[retriever]
type = SimpleIMAPSSLRetriever
server = imap.email.invalid
username = user@email.invalid
port = 993
password = password
mailboxes = ("INBOX", "Sent", "Spam")

[destination]
type = MDA_external
path = /usr/lib/dovecot/deliver
arguments = ("-e", "-m", "%(mailbox)")

[options]
read_all = false
delete_after = 30
delivered_to = false
received = true
verbose = 1
```

And finally start it with docker, or your container manager of choice (podman, kubernates etc.).

```bash
$ docker run -d -v /srv/mail/home:/home -v /srv/mail/cron.d:/etc/cron.d -v /srv/mail/ssl:/etc/ssl/private:ro -p 143 -p 993 -p 4190 --name mail ghcr.io/tctlrd/ddg6
```

Or use *docker-compose* (check out `docker-compose.example.yml`).

Users are created automatically with default password (`replaceMeNow`) on first start. To reset user passwords (of a running container):

```bash
$ docker exec -it mail passwd [user]
```

Sieve Filters
========

If you are using Sieve filters and want a `Refilter` mailbox to trigger their refiltering, create a refilter configuration per user (`/srv/mail/home/user/.getmail/getmailrc-refilter`):

```
# ~/.getmail/getmailrc-*: getmailrc refilter configuration

[retriever]
type = SimpleIMAPRetriever
server = localhost
port = 143
username = user
password = password
mailboxes = ("Refilter",)

[destination]
type = MDA_external
path = /usr/lib/dovecot/deliver
arguments = ("-e",)

[options]
read_all = false
delete = true
delivered_to = false
received = false
verbose = 1
```

Pull Requests are Welcome
========

Go ahead and improve this project and submit a pull request.

License
=======

This library is licensed under the [GNU Affero General Public License 3.0+](LICENSE_AGPL-3.0.txt) (AGPL-3.0+). Note that it is mandatory to make all modifications and complete source code of this library publicly available to any user.
