# Secure, Git-based Gentoo ebuild retrieval

```
  ______________________________________
/ This repo has been archived!           \
| Its successor is at:                   |
\ https://codeberg.org/duxsco/gentoo-git /
  --------------------------------------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||
```

Portage provides [three official ways](https://wiki.gentoo.org/wiki/Project:Portage/Repository_verification) to fetch Gentoo ebuilds. While `rsync` saves network traffic, `webrsync` supports downloads over HTTPS. The following Git-based approach tries to combine the efficiency of `rsync` with the security of `webrsync`.

Install and harden `dev-vcs/git`:

```bash
emerge --select --noreplace dev-vcs/git && \
git config --system includeIf.gitdir:/var/db/repos/gentoo/.path /etc/portage/gitconfig && \
git config --file /etc/portage/gitconfig http.sslCAInfo /etc/ssl/certs/4042bcee.0 && \
git config --file /etc/portage/gitconfig http.sslCAPath /etc/ssl/certs/4042bcee.0 && \
git config --file /etc/portage/gitconfig http.sslVersion tlsv1.3 && \
git config --file /etc/portage/gitconfig protocol.allow never && \
git config --file /etc/portage/gitconfig protocol.https.allow always; echo $?
```

The resulting configuration should look like:

```
➤ cat /etc/gitconfig
[includeIf "gitdir:/var/db/repos/gentoo/"]
	path = /etc/portage/gitconfig

➤ cat /etc/portage/gitconfig
[http]
	sslCAInfo = /etc/ssl/certs/4042bcee.0
	sslCAPath = /etc/ssl/certs/4042bcee.0
	sslVersion = tlsv1.3
[protocol]
	allow = never
[protocol "https"]
	allow = always
```

I assume that certificates for the Git repository are issued by Let's Encrypt. Thus, I only allow this single root CA:

```
➤ openssl x509 -noout -hash -subject -issuer -in /etc/ssl/certs/4042bcee.0
4042bcee
subject=C = US, O = Internet Security Research Group, CN = ISRG Root X1
issuer=C = US, O = Internet Security Research Group, CN = ISRG Root X1
```

[Enable git+https](https://wiki.gentoo.org/wiki/Project:Portage/Repository_verification):

```bash
mkdir -p /etc/portage/repos.conf && \
rsync -a /usr/share/portage/config/repos.conf /etc/portage/repos.conf/gentoo.conf && \
rsync -a /usr/share/portage/config/repos.conf /etc/portage/repos.conf/._cfg0000_gentoo.conf && \
sed -i \
    -e 's/^\(sync-type[[:space:]]*=[[:space:]]*\).*/\1git/' \
    -e 's#^\(sync-uri[[:space:]]*=[[:space:]]*\).*#\1https://anongit.gentoo.org/git/repo/sync/gentoo.git#' \
    -e '$ a sync-git-verify-commit-signature = yes' \
    /etc/portage/repos.conf/._cfg0000_gentoo.conf; echo $?
```

Execute `dispatch-conf` or `etc-update` and apply the changes to `/etc/portage/repos.conf/gentoo.conf` that way.

> ⚠️ For the (initial) clone, make sure that `* Trusted signature found on top commit` is printed out! ⚠️

As a last step, `/var/db/repos/gentoo` needs to be emptied and Gentoo ebuilds fetched:

```bash
find /var/db/repos/gentoo -maxdepth 1 -mindepth 1 -exec rm -rf {} + && \
chown portage:portage /var/db/repos/gentoo && \
emerge --sync
```

## Other Gentoo Linux repos

https://github.com/duxsco?tab=repositories&q=gentoo-
