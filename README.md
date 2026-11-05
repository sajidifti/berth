# berth

```text
┏┓ ┏━╸┏━┓╺┳╸╻ ╻
┣┻┓┣╸ ┣┳┛ ┃ ┣━┫   local Laravel sites
┗━┛┗━╸╹┗╸ ╹ ╹ ╹   Apache · PHP-FPM · Node · Composer
≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈
```

Serve local Laravel projects at `http://<name>.localhost` with Apache and PHP-FPM, choosing the PHP, Node and Composer version for each site.

berth is a single bash script. It installs what a site needs (the ondrej PHP PPA, PHP versions and extensions, Apache modules, nvm, Node and Composer), so one `berth host` can take a bare Ubuntu WSL distro to a working site.

```
$ cd /srv/sites/shop && berth host

  ┏┓ ┏━╸┏━┓╺┳╸╻ ╻
  ┣┻┓┣╸ ┣┳┛ ┃ ┣━┫   local Laravel sites v0.1.0
  ┗━┛┗━╸╹┗╸ ╹ ╹ ╹   Apache · PHP-FPM · Node · Composer
  ≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈

  Hosting shop
  http://shop.localhost  /srv/sites/shop

  Enter keeps the suggested version; type to change it.

  PHP       installed  8.2  8.3  8.5
            > 8.5
  Node      installed  18.20.8  24.15.0  26.13.0
            > 26
  Composer  installed  2.11.1
            > 2

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
- Composer 2
- nvm (`v0.41.0`)

Each step is skipped if it's already done, so running the install again updates berth and changes nothing else. To install a specific release instead of `main`, use the tag in the URL and in `BERTH_REF`:

```bash
curl -fsSL https://raw.githubusercontent.com/sajidifti/berth/v0.1.0/berth | BERTH_REF=v0.1.0 bash -s -- install
```

## Quick start

```
mv ~/projects/shop /srv/sites/       # or: git clone … /srv/sites/shop
cd /srv/sites/shop
berth host                           # answer or press Enter three times
berth php artisan migrate            # runs with the site's PHP
berth npm run dev                    # runs with the site's Node
```

Open `http://shop.localhost` in any browser.

## Commands

### Setup

| Command | What it does |
| --- | --- |
| `berth install` | Install or update berth itself, then prepare the machine (see above) |
| `berth php:install <v> [--ext a,b]` | Install PHP `<v>` (FPM, CLI, base extensions and any `--ext`) and its berth pool |
| `berth node:install <v>` | `nvm install <v>`, as you, without sudo |
| `berth composer:install [v]` | Download Composer `v` (default `2`) |

### Sites

| Command | What it does |
| --- | --- |
| `berth host [path]` | Serve the project at `path` (default: current folder), or reconfigure it if it's already hosted |
| `berth sites` | List sites: name, URL, PHP, Node, Composer, path |
| `berth rm <name>` | Remove the site's vhost and config entry |
| `berth doctor` | Read-only health check of Apache, every site and the PHP pools, with a fix for each problem |

`berth host` options:

| Option | Meaning |
| --- | --- |
| `--php 8.4` | PHP version (skips the question) |
| `--node 20` / `--node none` | Node version, or no Node |
| `--composer 2.2` | Composer version |
| `--name shop` | Site name. Default: the folder name. The URL is `http://<name>.localhost` |
| `--ext redis,imagick` | Extra PHP extensions |

### Run tools with the site's versions

```
berth php …        berth composer …
berth node …       berth npm …        berth npx …
```

berth walks up from the current folder to find the hosted project and uses its versions. Outside a hosted project, it uses the system `php`, berth's default Composer and nvm's default Node.

### Options for every changing command

- `--dry-run` lists what would be done and changes nothing.
- `--yes` skips the version questions and takes the suggested answers.

## Versions

`berth host` asks for PHP, Node and Composer versions. Each answer is pre-filled with a suggestion, so you can press Enter to keep it or edit it:

| | Suggested for a new site | Exceptions |
| --- | --- | --- |
| PHP | **8.5** | If `composer.json` `require.php` rules out 8.5, the closest installed version that fits |
| Node | **26** | `none` means the site gets no Node |
| Composer | **2** | `2.2` when PHP is below 7.3 |

For a site that's already hosted, the suggestion is its current version, so re-running `berth host` and pressing Enter changes nothing. Versions that aren't installed yet are installed as part of the run.

**PHP extensions:** berth always installs `cli fpm common mysql mbstring xml curl zip gd intl bcmath`. It adds any `ext-*` entries from `composer.json` `require`, plus `--ext`. Extensions built into PHP (`ctype`, `pdo`, `tokenizer` and so on) need nothing, and `pdo_mysql`, `dom`, `pdo_sqlite` and the like map to their packages.

**Composer versions:** `2` (latest 2.x), `2.2` (the long-term-support line for old PHP), `1`, or an exact version such as `2.7.9`. Each is a phar in `~/.local/share/berth/composer/`, checked against getcomposer.org's SHA-256.

## How it works

Every changing command first checks the current state and runs only the missing steps, printing each one (and every system command) as it goes. Running a command twice ends in "Nothing to do".

**PHP-FPM:** each PHP version gets a pool called `berth`, at `/etc/php/<v>/fpm/pool.d/berth.conf`. It runs as **your user** and listens on `/run/php/php<v>-berth.sock`. Laravel's `storage/` and `bootstrap/cache/` are written as you, so files created by artisan and by the web server never conflict.

**Apache:** one vhost per site, `/etc/apache2/sites-available/berth-<name>.conf`, with `ServerName <name>.localhost` and `*.<name>.localhost`, `DocumentRoot <project>/public`, `AllowOverride All` for Laravel's `.htaccess`, and PHP handed to that version's berth socket. berth runs `apache2ctl configtest` before every reload and rolls the vhost back if the test fails. Logs are `/var/log/apache2/berth-<name>-{error,access}.log`.

**Registry:** `~/.config/berth/sites`, one line per site, tab-separated: `name  path  php  node  composer`. Vhosts are generated from it, so don't edit the vhost files by hand; re-run `berth host` instead.

### Permissions

Apache runs as `www-data` and must be able to pass through every folder on the way to `public/`. Ubuntu home folders are `750`, so Apache can't reach projects in `~`. berth keeps projects in **`/srv/sites`**, owned by you, group `www-data`, mode `750`. Apache gets in through the group, other users don't, and you work there without sudo.

`berth host` refuses projects inside your home folder and prints the `mv` command to use instead. Tip: `ln -s /srv/sites ~/projects` keeps old paths working in your shell.

## Troubleshooting

Start with `berth doctor`. It lists each problem with the command that fixes it.

- **The site returns 500.** Check, in order:
  - `/var/log/apache2/berth-<name>-error.log`
  - `/var/log/php<v>-fpm.log`
  - the project's `storage/logs/laravel.log`

  A fresh Laravel app also needs a reachable database if it uses database sessions or cache.
- **Uploaded files return 403 after moving a project.** `public/storage` is an absolute symlink that still points into your home. Recreate it: `rm public/storage && berth php artisan storage:link`.
- **`mkdir` in `/srv` is denied.** That's normal: `/srv` belongs to root. Use `/srv/sites`, which `berth install` creates for you.
- **Protect secrets.** Project `.env` files are usually `644`. `chmod 600 .env` keeps them readable by you (and therefore by PHP), but not by `www-data`.

## Uninstall

```
berth rm <name>                       # for each site in `berth sites`
sudo rm /etc/php/*/fpm/pool.d/berth.conf && sudo systemctl restart 'php*-fpm'
rm -r ~/.local/bin/berth ~/.config/berth ~/.local/share/berth
```

PHP, Apache, nvm, Node and `/srv/sites` stay installed. Remove them separately if you want to.

## License

MIT. See [LICENSE](LICENSE).
