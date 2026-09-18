# Local Gmail IMAP Test Mailbox

This runs a private Dovecot IMAP server for testing against Google Takeout Gmail
archives. The service is cluster-internal only; use `kubectl port-forward` when
you want to connect a mail client or test app.

## Connect

```sh
kubectl -n mailbox port-forward svc/local-imap 1143:143
```

Then use these IMAP settings:

- Host: `127.0.0.1`
- Port: `1143`
- TLS: off
- Username: `test`
- Password: `changeme`

Change the password in `local-imap-users` before keeping real mail in the
cluster.

## Add A Gmail Takeout Mailbox

Google Takeout exports Gmail as one or more `.mbox` files. Put the extracted
Takeout folder on the Kubernetes node at:

```text
/mnt/gmail-takeout
```

The default importer expects this file:

```text
/mnt/gmail-takeout/Takeout/Mail/All mail Including Spam and Trash.mbox
```

Start a one-time import from the suspended CronJob:

```sh
kubectl -n mailbox create job \
  --from=cronjob/import-gmail-mbox \
  import-gmail-mbox-$(date +%s)
```

Watch the import:

```sh
kubectl -n mailbox logs -l job-name=<job-name> -f
```

The default import target is an IMAP folder named `Gmail`.
