# Uptime
Extremely simple, static, server-free uptime monitoring.

If you have GitHub or a server to run this on already, there's no additional procurement needed.


## What problem does this solve?

This monitors the uptime/downtime status of a single website, without:

- procuring and setting up a commercial monitoring solution
- introducing security risks involved with additional running servers. An [ATO](https://digital.gov/resources/an-introduction-to-ato/) may still be needed, but we expect it to be minimal.
- hosting a server
- running a database

## How does it work?

This monitors a single site and records the uptime/downtime in a log. It displays the last 65 events, showing the uptime/downtime in a simple chart.

Process:

- Every fifteen minutes, a scheduled task checks the status of the website. (By default, we use GitHub Actions. You can also follow the instructions for setting this up on your own server.)
- The task writes the time, HTTP status code, and up/down status in a log, `log.txt`. The write to `log.txt` is append-only, so we can track uptime over all time. On GitHub, if the task runs every 15 minutes, it will run for 40 years without running into GitHub file size limits.
- After the write to `log.txt`, the task copies the last 65 events to a separate log, `readlog.txt`. This smaller file is what the HTML page uses to display recent activity.
- If using GitHub, the GitHub Action commits these file changes, saving the data. Then, GitHub redeploys the GitHub Pages site, showing the new data. When running this on a server, this commit step isn't necessary.


## Installation

First, update the `settings.json` to configure your Uptime monitor.

- Website
  - name: The full name of your website
  - short_name: The name of your website that will appear in the tab title
  - url: The URL you'd like to monitor.
- Status checks
  - refresh_every: The number of minutes between page refreshes. This is different than the number of minutes between each status check. Default: 1.
  - recent_entries: The number of status bars to display.
  - warn_after: After this number of minutes, show a message that prompts the user to check the logs of the status check jobs. When using GitHub Actions, there's sometimes a delay, so jobs set to run every 15 minutes might run every 20-25 minutes. This isn't necessarily an error, so we suggest alerting at or after 25 minutes.
- Logs
  - Paths
    - all: The file path to the log of all the status check records. Default: log.txt
    - recent: The file path to the recent status check records to display. Default: readlog.txt
  - Links
    - all: A public URL or file path to the log of all the status check records. Viewers will be able to navigate to this link.
    - actions: A public URL or file path to where the server check jobs are being recorded. For example, if you are using GitHub Actions, this would link to the GitHub Actions logs.


Then, follow the setup instructions for your platform below, either **GitHub Actions** or **Windows server**.

### Using GitHub Actions

- First, make sure GitHub Actions are allowed for your GitHub account or organization, and that Actions can have Write access to repositories.
- Clone this repository to your own GitHub account / organization. (We recommend cloning because forking appears to cause problems with GitHub Actions.)
- Delete the contents of `log.txt` and `readlog.txt`, but leave the empty files in place.
- Change the URL in `.github/workflows/ping.yaml` to the URL you want to monitor. This should match the website URL setting in `settings.json`.
- Go to the GitHub Pages settings for your repository. Set it to deploy your `main` branch at `/` (root).

The GitHub Pages page will update a few minutes after every server status check.

To run the server status check manually:

- Go to the Actions tab.
- In the left sidebar, select "Check server status".
- In the light blue bar, there will be a "Run workflow" dropdown.
- Click it, make sure the branch is "main", and click "Run workflow".

This will gather your first bit of data, and will force the GitHub Pages page to deploy.


### Using a Windows server

You'll have to set up the web hosting, server status check, and email notifications yourself.

We've translated the GitHub Actions commands to Windows PowerShell version 7.0.

#### Web hosting

Clone the repository and place it into a directory that can be served by a webserver.

#### Server check and email notifications

Using Task Scheduler or equivalent, schedule the following server status check script to run every 15 minutes, indefinitely.

```sh
# Check the server status and write it to the log files

$now=Get-Date -Format "o"
$httpCode=$(curl -S -s -o NUL -w "%{http_code}" --connect-timeout 30 --max-time 60 FILL_IN_MONITORED_URL)
$statusText = If (($httpCode -eq 200) -or ($httpCode -eq 302)) {"up"} Else {"dn"}
echo "$now | $httpCode | $statusText" | Out-File -FilePath \absolute\path\to\log.txt -Encoding utf8 -Append
Get-Content \absolute\path\to\log.txt -Tail 65 | Out-File -FilePath \absolute\path\to\readlog.txt -Encoding utf8

# example usage of Did-Status-Change:
#   @("up" | "dn") | Did-Status-Change
#   #=> True
#
#   @("up" | "up") | Did-Status-Change
#   #=> False
#
function Did-Status-Change() {
   [cmdletbinding()]
    param (
        [Parameter(Mandatory = $true, ValueFromPipeline=$true)]
        [string[]] $InputArray
    )
    ($input[0] -ne $input[1])
}

# Determine if the server status changed (from up to down, or down to up)
$didStatusChange = Get-Content \absolute\path\to\readlog.txt -Tail 2 | ForEach-Object { $_.split("|")[2].Trim() } | Did-Status-Change

If ($didStatusChange) {
  # Get the last status
  $lastStatus = Get-Content \absolute\path\to\readlog.txt -Tail 1 | ForEach-Object { $_.split("|")[2].Trim() }
  # Convert dn -> down, up -> up
  $lastStatusFull = If ($lastStatus -eq "up") {"up"} Else {"down"}
  # Send an email alert with the appropriate status
  # TODO: This mail command may not be exactly right.
  SendMail \
    -From TBD \
    -To people who need to be alerted \
    -Subject FILL IN WEBSITE NAME is $lastStatusFull\
    -Body FILL IN WEBSITE NAME is $lastStatusFull.<br/><br/><a href="FILL IN MONITOR SITE URL">View the uptime monitor</a>
}
```

## Running the server (Unix / macOS)

To run the site locally, first clone the repository, `cd` into the folder. Then, run a simple HTTP server to serve this directory. If you have Python 3 installed, run the following (your command may use either `python` or `python3`:

```sh
python3 -m http.server
```

This will run a simple server to serve the static page. Visit [localhost:8000](http://localhost:8000) to see the page.

### Adding log data

If you need log data, just run the following to simulate some uptime and downtime.

```sh
count=60
for i in $(seq $count); do
  echo "$(TZ=US/Eastern date -Iseconds) | 200 | up" >> log.txt
done

count=20
for i in $(seq $count); do
  echo "$(TZ=US/Eastern date -Iseconds) | 500 | dn" >> log.txt
done
```

Then run:

```sh
tail -n 65 log.txt > readlog.txt
```

You should see the status information when you refresh. Remember not to commit these changes, otherwise you'll have fake data in your log history.

