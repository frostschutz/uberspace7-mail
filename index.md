[Uberspace](https://github.com/Uberspace) is a hosting provider for web and mail.

You can set up any number of [mail domains](https://manual.uberspace.de/mail-domains.html) and [mailboxes](https://manual.uberspace.de/mail-mailboxes.html) as described in the [Uberspace 7 manual](https://manual.uberspace.de).

Unfortunately, this comes with a limitation:

> You can use mailboxes in the form of `$MAILBOX@$USER.uber.space`. 
>
> If you have set up additional domains, `$MAILBOX@$DOMAIN` will also work.

So any mailbox you define will be available under any domain.

Creating domain-specific mail addresses (`isabell@philae.tld` but not `isabell@obelisk.tld`) is not possible.

Unless you implement it yourself. :-)

----

Uberspace uses netqmail's [.qmail files](https://web.archive.org/web/20200902011211/https://wiki.uberspace.de/mail:dotqmail) in the user's home directory to handle incoming mail.

qmail provides the address the mail was sent to in two environment variables `EXT` and `HOST`, so the full address is `$EXT@$HOST`.

qmail then starts `vdeliver` (if spamfolder is disabled) or `maildrop ~/.spamfolder` (if spamfolder is enabled) and these programs then place the mail into the mailbox specified as `EXT`.

> `$ uberspace mail spamfolder status`  
> Your `/home/isabell/.qmail-default` contains things different from our default setup. Therefore we cannot tell about the status of the spam folder feature.  
> Our standard setup consists of a `/home/isabell/.qmail-default` file containing the string `|/usr/bin/vdeliver` (spam folder disabled) or `|maildrop /home/isabell/.spamfolder` (spam folder enabled).

As for `HOST`, it is simply ignored in the default setup of Uberspace. So `isabell@anywhere` always goes into the `isabell` mailbox, regardless which domain it was sent to.

----

Fortunately, you can edit your `.qmail-default` file to run a custom script instead.

    |/home/isabell/bin/u7mail

In this script, you can change the mailbox easily by setting `EXT` to a different value.

A few examples:

* Deliver all mail to `isabell`:

  ```bash
  #!/bin/bash
  EXT=isabell /usr/bin/vdeliver
  ```
    
* Deliver only `isabell@philae.tld` to `isabell`, silently discard everything else:

  ```bash
  #!/bin/bash
  if [ "${EXT,,}@${HOST,,}" == "isabell@philae.tld" ]
  then
      EXT=isabell /usr/bin/vdeliver
  fi
  exit 0 # discard
  ```
    
* Arbitrary address to mailbox routing:
  
  ```bash
  #!/bin/bash
  address="${EXT,,}@${HOST,,}"
  case "${address}" in
      # philae.tld
      "isabell@philae.tld")      EXT=isabell /usr/bin/vdeliver  ;;
      "isabell.doe@philae.tld")  EXT=isabell /usr/bin/vdeliver  ;;
      # obelisk.tld
      *"@obelisk.tld")  EXT=rosetta /usr/bin/maildrop ~/.spamfolder  ;;
      # catchall
      *)  EXT=default /usr/bin/vdeliver  ;;
  esac
  # default:
  /usr/bin/vdeliver
  ```

In this fashion, you can add an additional layer between mailboxes and mail addresses.

You can define any number of mail addresses, mail aliases, selective catch-alls.

You can create mailboxes that never receive mail (for sending only), or choose arbitrary mailbox names completely unrelated to their addresses (prevents login attempts, security by obscurity).

By reading mail headers and body from stdin, you could even filter and e.g. deliver spam to a dedicated spam mailbox — however, at this point you might want to consider writing a [custom maildrop filter](https://web.archive.org/web/20200902011156/https://wiki.uberspace.de/mail:maildrop) instead of a shell script.

----

##### CAVEATS

* This method is not officially supported in any way. Expect it to break at any time. *Please don't bother Uberspace support with it.*

  - Delivering mail to a mailbox with wrong filenames or permissions can render the mailbox inaccessible (protocol errors when trying to access mail). That's why the delivery itself is still left to `vdeliver` or `maildrop`.

* A single error in your script might end up *silently discarding mails*, or delivering mangled ones. You have been warned.

  - Use `.qmail-isabell` instead of `.qmail-default` for a script that only affects `isabell@$HOST` addresses.

  - Use `.qmail-test-default` instead of `.qmail-default` for testing new / changed scripts (with `test-thing@philae.tld` address).
  
  - Use the [unofficial bash strict mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/):

    ```bash
    #!/bin/bash
    set -euo pipefail
    IFS=$'\n\t'
    ```

* [Accessing your Mails](https://manual.uberspace.de/mail-access.html) is unaffected, so only `$MAILBOX@$DOMAIN` works. This makes it inferior to Uberspace 6 namespaces, which are no longer available under Uberspace 7.

  - `isabell@obelisk.tld` is still Isabell's mailbox, even if you mapped it to Rosetta.
 
  - `$DOMAIN` must have a [MX record](https://en.wikipedia.org/wiki/MX_record) like `MX -> philae.uberspace.de` (your host name). Furthermore, it must be the primary MX record for this domain. This can be a problem when migrating mail servers. Changing the MX record makes your mailbox inaccessible, while mails are still delivered to it for another 24-48 hours.
    
    * Instead of `isabell@yourdomain.tld`, use `isabell@isabell.philae.uberspace.de` (your internal Uberspace 7 maildomain) when accessing the mailbox.
    
    * Alternatively, use a custom subdomain like `uberspace mail domain add u7mail.yourdomain.tld` with MX record to access your mails (does not reveal Uberspace username and allows migrating).
