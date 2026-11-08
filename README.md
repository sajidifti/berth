# berth

```text
┏┓ ┏━╸┏━┓╺┳╸╻ ╻
┣┻┓┣╸ ┣┳┛ ┃ ┣━┫   local Laravel & PHP sites
┗━┛┗━╸╹┗╸ ╹ ╹ ╹   Apache · PHP-FPM · Node · Composer
≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈
```

Serve local Laravel and plain PHP projects at `http://<name>.localhost` with Apache and PHP-FPM, choosing the PHP, Node and Composer version for each site.

berth is a single bash script. It installs what a site needs (the ondrej PHP PPA, PHP versions and extensions, Apache modules, nvm, Node and Composer), so one `berth site:host` can take a bare Ubuntu machine or WSL distro to a working site.

```
$ cd /srv/sites/shop && berth site:host

  ┏┓ ┏━╸┏━┓╺┳╸╻ ╻
  ┣┻┓┣╸ ┣┳┛ ┃ ┣━┫   local Laravel & PHP sites v0.2.0
  ┗━┛┗━╸╹┗╸ ╹ ╹ ╹   Apache · PHP-FPM · Node · Composer
  ≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈

  Hosting shop
  http://shop.localhost  /srv/sites/shop
  Laravel app · serving public/

  Enter keeps the suggested version; type to change it.

  PHP       installed  8.2  8.3  8.5
            > 8.5
  Node      installed  18.20.8  24.15.0  26.13.0
            > 26
  Composer  installed  2.11.1
            > 2
  HTTPS     also serve https://shop.localhost?
            > [y/N]

  [1/3] write and enable vhost /etc/apache2/sites-available/berth-shop.conf (http://shop.localhost → PHP 8.5)
      + sudo tee /etc/apache2/sites-available/berth-shop.conf
      + sudo a2ensite -q berth-shop
  [2/3] record shop in ~/.config/berth/sites
  [3/3] apache2ctl configtest, then reload Apache
      + sudo systemctl reload-or-restart apache2

  ✓ shop is live at http://shop.localhost
    PHP 8.5 · Node 26 · Composer 2
    Set APP_URL=http://shop.localhost in .env
    Run tools with these versions: berth php artisan …, berth composer …, berth npm …
```

## Requirements

- Ubuntu (tested on 24.04 under WSL2 with systemd). Other distros are refused.
- A user with `sudo`. berth runs as you and asks for sudo only for system steps.
- On WSL, mirrored networking (`networkingMode=mirrored` in `%USERPROFILE%\.wslconfig`) lets Windows browsers open `http://<name>.localhost` with no hosts-file edits.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/sajidifti/berth/main/berth | bash -s -- install
```

To read the script before running it:

```bash
curl -fsSLO https://raw.githubusercontent.com/sajidifti/berth/main/berth
less berth
bash berth install
```

Or from a clone:

```bash
git clone https://github.com/sajidifti/berth.git && cd berth && ./berth install
```

`berth install` puts the script at `~/.local/bin/berth`, which must be on your `PATH`. It then prepares the machine:

- the ondrej/php PPA
- Apache with `mpm_event`, `proxy_fcgi`, `setenvif` and `rewrite` (any `mod_php` is disabled)
- `/srv/sites` for your projects
- nvm (`v0.41.0`)
- PHP, Node and Composer, if they're missing (see [Default versions](#default-versions))

Each step is skipped if it's already done, so running `berth install` again only repairs what's missing. To install a specific release instead of `main`, use the tag in the URL and in `BERTH_REF`:

```bash
curl -fsSL https://raw.githubusercontent.com/sajidifti/berth/v0.2.0/berth | BERTH_REF=v0.2.0 bash -s -- install
```

### Update

```bash
berth update                  # latest main
berth update --ref v0.2.0     # a specific release
```

`berth update` downloads berth from GitHub and replaces `~/.local/bin/berth`, showing the old and new version. It changes nothing else.

## Quick start

```
mv ~/projects/shop /srv/sites/       # or: git clone … /srv/sites/shop
cd /srv/sites/shop
berth site:host                      # press Enter to take each suggestion
berth php artisan migrate            # runs with the site's PHP
berth npm run dev                    # runs with the site's Node
```

Open `http://shop.localhost` in any browser.

### Plain PHP apps

`berth site:host` works the same way for a project that isn't Laravel. It finds the folder to serve (the web root) by looking for `index.php` or `index.html`, in this order:

1. `public/`
2. the project folder itself
3. `web/`, `public_html/`, `htdocs/`, `www/`, `html/`

```
mkdir /srv/sites/blog && echo '<?php phpinfo();' > /srv/sites/blog/index.php
cd /srv/sites/blog && berth site:host    # → http://blog.localhost
```

Use `--root <folder>` to serve some other folder, for example `berth site:host --root dist`. The folder must be inside the project. `.htaccess` files work as usual. When the project folder itself is the web root, berth blocks web access to `.git`, `.svn`, `.hg` and `.env*`.

### HTTPS

`berth site:host` asks whether to also serve the site over HTTPS. The answer defaults to no. Pressing `y` gives you `https://<name>.localhost`, and `http://` keeps working alongside it.

To switch later, run `berth site:host` in the project again and answer the question, where Enter keeps the current state. `--ssl` and `--no-ssl` skip the question.

The first time HTTPS is turned on, berth creates one local certificate authority (CA) and adds it to every certificate store it finds on the machine:

| Store | Used by | Needs |
| --- | --- | --- |
| Ubuntu's certificate store | `curl`, PHP, git | sudo |
| Browser databases in your home folder | Chrome, Chromium and Firefox on Ubuntu, snaps included | nothing |
| Your Windows user's store (WSL only) | Chrome and Edge on Windows | one Windows Yes/No prompt, no admin rights |

Other details:

- The CA can only sign names under `.localhost`, so it can't be used to impersonate a real domain. Its key is at `~/.local/share/berth/ssl/ca.key`, readable only by you.
- Installed a browser after setting up HTTPS? Run `berth ssl:install` again and it fills in what's missing.
- Firefox on Windows may keep its own list. If it warns about the certificate, import `ca.crt` under Settings > Certificates > Authorities.
- Node doesn't read Ubuntu's certificate store. For a Node script that calls these URLs, set `NODE_EXTRA_CA_CERTS=~/.local/share/berth/ssl/ca.crt`.
- Site certificates last 825 days. Running `berth site:host` renews one that has under 30 days left, and `berth doctor` reports it.
- berth never edits `.env`. After turning HTTPS on or off, update `APP_URL` yourself.

## Commands

Every command follows one of three patterns:

| Pattern | For | Examples |
| --- | --- | --- |
| `berth <command>` | berth itself and the machine | `install`, `update`, `doctor` |
| `berth <thing>:<action>` | sites and versions berth manages | `site:host`, `php:install`, `node:list` |
| `berth <tool> …` | running a tool with the site's versions | `berth php artisan migrate`, `berth npm run dev` |

Sites use `host`, `list` and `remove`; PHP, Node and Composer use `install`, `list`, `update`, `default` and `remove`; HTTPS uses `install`, `list` and `remove`.

### berth

| Command | What it does |
| --- | --- |
| `berth install` | Install berth, then prepare the machine (see above) |
| `berth update [--ref tag]` | Update berth itself from GitHub (see [Update](#update)) |
| `berth doctor` | Read-only health check of Apache, every site and the PHP pools, with a fix for each problem |
| `berth help`, `berth version` | Usage and version |

### Sites

| Command | What it does |
| --- | --- |
| `berth site:host [path]` | Serve the project at `path` (default: current folder), or reconfigure it if it's already hosted |
| `berth site:list` | List sites: name, URL, PHP, Node, Composer, path, web root |
| `berth site:remove <name>` | Remove the site's vhost and config entry |

`berth site:host` options:

| Option | Meaning |
| --- | --- |
| `--php 8.4` | PHP version (skips the question) |
| `--node 20` / `--node none` | Node version, or no Node |
| `--composer 2.2` | Composer version |
| `--name shop` | Site name. Default: the folder name. The URL is `http://<name>.localhost` |
| `--root web` | Folder to serve, relative to the project (`.` for the project folder). Default: detected; see [Plain PHP apps](#plain-php-apps) |
| `--ext redis,imagick` | Extra PHP extensions |
| `--ssl` / `--no-ssl` | Turn HTTPS on or off (skips the question). See [HTTPS](#https) |

### HTTPS commands

| Command | What it does |
| --- | --- |
| `berth ssl:install` | Create the local CA if needed and add it to any store that's missing it. `site:host` does this for you the first time |
| `berth ssl:list` | Where the CA is trusted, and each HTTPS site's certificate expiry |
| `berth ssl:remove` | Remove the CA from every store and delete it. Refused while any site uses HTTPS |

### PHP, Node and Composer

| Command | What it does |
| --- | --- |
| `berth php:install <v> [--ext a,b]` | Install PHP `<v>` (FPM, CLI, base extensions and any `--ext`) and its berth pool |
| `berth php:list` | Installed PHP versions with their release, which one is the system `php`, and the sites using each. Also lists versions the PPA offers |
| `berth php:default [v]` | Make PHP `v` the default: new sites suggest it, and the system `php` command switches to it (`update-alternatives`, needs sudo). Without `v`, shows the default |
| `berth php:update [v]` | Upgrade PHP `v` to its newest patch release through apt, then restart its FPM. Without `v`, every installed version |
| `berth php:remove <v>` | `apt-get remove` PHP `<v>`'s packages and delete its berth pool |
| `berth node:install <v>` | `nvm install <v>`, as you, without sudo |
| `berth node:list` | Installed Node versions, nvm's default and the sites using each |
| `berth node:default [v]` | Make Node `v` the default and nvm's default, so new shells use it |
| `berth node:update [v]` | Install the newest release of Node line `v` (e.g. `24`), carrying over global npm packages. Without `v`, every installed line |
| `berth node:remove <v>` | `nvm uninstall <v>` |
| `berth composer:install [v]` | Download Composer `v` (default `2`) |
| `berth composer:list` | berth's Composer versions and the sites using each, plus any system `composer` |
| `berth composer:default [v]` | Make Composer `v` the default and point `~/.local/bin/composer` at it, so plain `composer` runs it |
| `berth composer:update [v]` | Download the newest release of Composer line `v` (`2`, `2.2` or `1`). Without `v`, every installed line |
| `berth composer:remove <v>` | Delete Composer `<v>`'s phar |

Updates stay on a line: `php:update 8.4` never moves you to 8.5, and `node:update 24` stays on 24.x. To move a site to another version, use `site:host --php 8.5`. Exact versions (`node:install 24.15.0`, `composer:install 2.7.9`) are pinned and have nothing to update. After `node:update`, sites set to `24` use the new release right away. The old release stays installed until you run `berth node:remove <version>`.

berth won't remove a version that a site or the default uses. It names the sites, and you move each one to another version (`berth site:host --php 8.5`) or remove it (`berth site:remove`) first.

`php:remove` asks before it removes anything. It also warns when:

- apt would remove other packages that depend on that PHP (phpMyAdmin, for example)
- Apache's own `php<v>-fpm` conf is enabled, so pages outside berth would stop running PHP
- the version is the system `php` command

The settings in `/etc/php/<v>` stay; `sudo apt purge 'php<v>-*'` deletes them. With `--yes`, `php:remove` skips the question, but it still refuses if apt would remove packages other than PHP `<v>`'s own.

### Run tools with the site's versions

```
berth php …        berth composer …
berth node …       berth npm …        berth npx …
```

berth walks up from the current folder to find the hosted project and uses its versions. Outside a hosted project, it uses the default versions.

Watch the colon: `berth composer:update` updates Composer itself, while `berth composer update` runs Composer's `update` in your project and rewrites `composer.lock`. The same goes for every tool: `<tool>:<action>` manages the tool and `<tool> …` runs it.

### Options for every changing command

- `--dry-run` lists what would be done and changes nothing.
- `--yes` skips the questions and takes the suggested answers. For HTTPS, that's off for a new site and the current state for an existing one.

## Versions

`berth site:host` asks for PHP, Node and Composer versions. Each answer is pre-filled with a suggestion, so you can press Enter to keep it or edit it:

| | Suggested for a new site | Exceptions |
| --- | --- | --- |
| PHP | the default PHP | If `composer.json` `require.php` rules it out, the closest installed version that fits |
| Node | the default Node | `none` means the site gets no Node |
| Composer | the default Composer | `2.2` when PHP is below 7.3 |

For a site that's already hosted, the suggestion is its current version, so re-running `berth site:host` and pressing Enter changes nothing. Versions that aren't installed yet are installed as part of the run.

### Default versions

Each tool has one default version. New sites suggest it, and outside a hosted project `berth php`, `berth node` and `berth composer` use it. The default is also the version your shell runs:

| Default | Plain command in your shell |
| --- | --- |
| PHP | `php` (system-wide, through `update-alternatives`) |
| Node | `node`, `npm` (nvm's default, in new shells) |
| Composer | `composer` (through `~/.local/bin/composer`) |

**`berth install` never switches a tool you already have.** For each tool:

- **Already installed:** berth records what you use as its default. PHP is the current `php` command, Node is nvm's default, and Composer is an existing berth copy.
- **Missing:** berth installs the newest version and makes it the default. PHP is the newest stable release from php.net, Node is the newest LTS, and Composer is the newest 2.x.

To change a default, run `berth php:default 8.4`, `berth node:default 22` or `berth composer:default 2.2`. Each one installs the version if it's missing. Run any of them without a version to see the current default. Defaults are stored in `~/.config/berth/config`.

Two cases where a plain command won't follow the default:

- **Composer:** if `~/.local/bin/composer` already exists and isn't berth's, berth leaves it alone. If another `composer` comes earlier on your `PATH`, berth tells you which one wins.
- **PHP:** `php:default` changes `php` for every user on the machine, since that's how Ubuntu's `update-alternatives` works.

**PHP extensions:** berth always installs `cli fpm common mysql mbstring xml curl zip gd intl bcmath`. It adds any `ext-*` entries from `composer.json` `require`, plus `--ext`. Extensions built into PHP (`ctype`, `pdo`, `tokenizer` and so on) need nothing, and `pdo_mysql`, `dom`, `pdo_sqlite` and the like map to their packages.

**Composer versions:** `2` (latest 2.x), `2.2` (the long-term-support line for old PHP), `1`, or an exact version such as `2.7.9`. Each is a phar in `~/.local/share/berth/composer/`, checked against getcomposer.org's SHA-256.

## How it works

Every changing command first checks the current state and runs only the missing steps, printing each one (and every system command) as it goes. Running a command twice ends in "Nothing to do".

**PHP-FPM:** each PHP version gets a pool called `berth`, at `/etc/php/<v>/fpm/pool.d/berth.conf`. It runs as **your user** and listens on `/run/php/php<v>-berth.sock`. Laravel's `storage/` and `bootstrap/cache/` are written as you, so files created by artisan and by the web server never conflict.

**Apache:** one vhost per site, `/etc/apache2/sites-available/berth-<name>.conf`, with `ServerName <name>.localhost` and `*.<name>.localhost`, `DocumentRoot` set to the web root (`<project>/public` for Laravel), `AllowOverride All` so `.htaccess` files work, and PHP handed to that version's berth socket. A site with HTTPS gets a second, matching vhost on port 443, with its certificate in `/etc/apache2/berth-ssl/`. berth runs `apache2ctl configtest` before every reload and rolls the vhost back if the test fails. Logs are `/var/log/apache2/berth-<name>-{error,access}.log`.

**Registry:** `~/.config/berth/sites`, one line per site, tab-separated: `name  path  php  node  composer  root  ssl`. Vhosts are generated from it, so don't edit the vhost files by hand; re-run `berth site:host` instead.

### Permissions

Apache runs as `www-data` and must be able to pass through every folder on the way to the web root. Ubuntu home folders are `750`, so Apache can't reach projects in `~`. berth keeps projects in **`/srv/sites`**, owned by you, group `www-data`, mode `750`. Apache gets in through the group, other users don't, and you work there without sudo.

`berth site:host` refuses projects inside your home folder and prints the `mv` command to use instead. Tip: `ln -s /srv/sites ~/projects` keeps old paths working in your shell.

## Troubleshooting

Start with `berth doctor`. It lists each problem with the command that fixes it.

- **The site returns 500.** Check, in order:
  - `/var/log/apache2/berth-<name>-error.log`
  - `/var/log/php<v>-fpm.log`
  - for Laravel, the project's `storage/logs/laravel.log`

  A fresh Laravel app also needs a reachable database if it uses database sessions or cache.
- **Uploaded files return 403 after moving a project.** `public/storage` is an absolute symlink that still points into your home. Recreate it: `rm public/storage && berth php artisan storage:link`.
- **`mkdir` in `/srv` is denied.** That's normal: `/srv` belongs to root. Use `/srv/sites`, which `berth install` creates for you.
- **The browser warns about the HTTPS certificate.** Run `berth ssl:list` to see where the CA is missing, then `berth ssl:install`, and restart the browser. If Windows didn't add the CA, you may have declined its prompt (it can open behind the terminal), or your organisation may block new root certificates.
- **Protect secrets.** Project `.env` files are usually `644`. `chmod 600 .env` keeps them readable by you (and therefore by PHP), but not by `www-data`.

## Uninstall

```
berth site:remove <name>              # for each site in `berth site:list`
berth ssl:remove                      # if you used HTTPS
sudo rm /etc/php/*/fpm/pool.d/berth.conf && sudo systemctl restart 'php*-fpm'
rm -r ~/.local/bin/berth ~/.config/berth ~/.local/share/berth
```

PHP, Apache, nvm, Node and `/srv/sites` stay installed. Remove them separately if you want to.

## License

MIT. See [LICENSE](LICENSE).
