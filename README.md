# Secure, Git-based Gentoo ebuild retrieval

> ⚠️ The following solution might not be desired on squashfs/overlayfs systems due to additional disk writes a Git hard reset incurs! ⚠️

Portage provides [three official ways](https://wiki.gentoo.org/wiki/Project:Portage/Repository_verification) to fetch Gentoo ebuilds. While `rsync` saves network traffic, `webrsync` supports downloads over HTTPS. The following Git-based approach tries to combine the efficiency of `rsync` with the security of `webrsync`.

First, we have to patch `sys-apps/portage` in order to avoid running in following problem that may occur during the merge reset after the shallow fetch:

![Git issue](assets/issue.png)

Verify the patch (copy&paste one after the other):

```bash
# Fetch the public key; ADJUST THE MAIL ADDRESS!
gpg --auto-key-locate clear,dane,wkd,hkps://keys.duxsco.de --locate-external-key d at "my github username" dot de

gpg --verify git.patch.sha512.asc git.patch.sha512

sha512sum -c git.patch.sha512
```

[Save the patch](https://wiki.gentoo.org/wiki//etc/portage/patches) and rebuild `sys-apps/portage`:

```bash
mkdir -p /etc/portage/patches/sys-apps/portage && \
rsync -av git.patch /etc/portage/patches/sys-apps/portage/ && \
chown root: /etc/portage/patches/sys-apps/portage/git.patch && \
emerge -1 sys-apps/portage
```

Install and harden `dev-vcs/git`:

```bash
emerge --noreplace dev-vcs/git && \
git config --system includeIf.gitdir:/var/db/repos/gentoo/.path /etc/portage/gitconfig && \
git config --system --add safe.directory /var/db/repos/gentoo && \
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
[safe]
	directory = /var/db/repos/gentoo

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
    -e '$ a clone-depth = 1' \
    -e '$ a sync-depth = 1' \
    /etc/portage/repos.conf/._cfg0000_gentoo.conf; echo $?
```

Execute `dispatch-conf` or `etc-update` and apply the changes to `/etc/portage/repos.conf/gentoo.conf`.

As a last step, `/var/db/repos/gentoo` needs to be emptied and Gentoo ebuilds fetched:

```bash
find /var/db/repos/gentoo -maxdepth 1 -mindepth 1 -exec rm -rf {} + && \
chown portage:portage /var/db/repos/gentoo && \
qlist --installed app-portage/eix >/dev/null 2>&1 && \
eix-sync || \
emerge --sync
```

## Other Gentoo Linux repos

https://github.com/duxsco?tab=repositories&q=gentoo-
