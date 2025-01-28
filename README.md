# GoCardless-to-CSV

# What is this? 

In Europe, GoCardless allows you to use OpenBanking to fetch transaction statements from your bank in a single unified format.
It supports more than 2600+ banking institutions, and for private individuals, its use is free (as of 2025).

This plain text accounting tool allows you to use GoCardless API to fetchning statements of your transactions from any institution that GoCardless supports.

# How does it work

- you create GoCardless account

- you set up this tool with GoCardless API keys

- you run `gocardless-to-csv setup` and choose your institution, and instruct GoCardless to ask it via OpenBanking for access to your transaction history, which establishes so-called "link"

- from now on, you can use this link to fetch transaction statements, and save them as JSON

- tool then allows you to convert GoCardless JSON to CSV file suitable for ingestion into hledger/ledger

# Limitations

As of 2025, GoCardless is mostly limited to Europe, and most notably does not support US.

GoCardless has pretty severe API usage limits - you can run `fetch` at most 4 times per 24 hours, and `link` at most 50 institutions.

Lots of banks allow access to just last 90 days of transactions.

Obviously, you need to be happy with GoCardless seeing your accounts and transactions, and your bank knowing that you've enabled this access.

# Installation

Clone this repository and run `pip install -r requirements.txt`.

Tool depends on `nordigen-python` which is a client library for GoCardless API, and `pyfzf` that is used to provide a selector for
banking institutions (if you have `fzf` command line tool installed). You can remove `pyfzf` from requirements.txt if you dont need this or you don't have `fzf` installed.

# Initial Setup

Go to https://bankaccountdata.gocardless.com/overview/ and register your account

Then go to https://bankaccountdata.gocardless.com/user-secrets/ and create a new secret. Call it `gocardless-to-csv` (or anything you like), and write down "secret ID" and "secret key"

Create a config file (default location is in `~/.config/gocardless-to-csv/config.ini`) and put the ID and key there:

```
[nordigen]
SECRET_ID=<...>
SECRET_KEY=<...>
```

You can now start to set up links to your banks

# Linking to GoCardless Sandbox

GoCardless provides "developer sandbox" with a pre-made link to a toy "bank" that has two accounts full of synthetic transactions.

You can try the tool with this sandbox, by running `gocardless-to-csv setup --sandbox`

I've included an example of Sandbox transaction file and its conversion to CSV in the `examples` folder.

# Linking your bank(s)

Run `gocardless-to-csv setup`, enter your country code and then select your institution from the list.

Tool will print out the web link that you need to visit to complete the linking process. This will redirect you to your bank,
where you can authenticate and confirm that you do want to give GoCardless access to your transactions.

By default, GoCardless requests access to last 90 days of transactions, and this access is valid for 90 days. Some banks
allow you to request more history, and the tool will always try to request the maximum possible amount of history days.

For every link you set up, tool will ask you for your own identifier for this link. This is needed for configuration, and to allow
you to sever the link before it expires on its own.

# Configuring accounts

Once you have established some links, run `gocardless-to-csv list`. This will print out the GoCardless IDs for all the accounts 
visible through every link, for example:

```
> gocardless-to-csv.py list
sandbox (status=LN):
  account: 8fa97631-ec2e-4f70-813d-1fbd5c5dc3dc
  account: f01aeb75-128a-46dc-997c-b2ddfd5edf74
```

You can associate these account names with human-friendly file names that would be used to store the transaction from these accounts.
Add the account ids as sections to `config.ini` and specify `file` for them:

```
[8fa97631-ec2e-4f70-813d-1fbd5c5dc3dc]
file=%%Y/sandbox1.json

[f01aeb75-128a-46dc-997c-b2ddfd5edf74]
file=%%Y/sandbox2.json
```

You can include `strftime`-compatible format specifiers in the filename (but use two `%%`, not one), and they would be automatically
filled out using the start date of the fetch, and all the necessary directories will be created if they dont exist.

# Fetching transactions

Run `gocardless-to-csv fetch` to fetch all account from all links, or `gocardless-to-csv fetch --ref <your link reference>` to
fetch only the accounts from the named link. 

You can restrict the time period with `--start YYYY-MM-DD` and `--end YYYY-MM-DD`, or use `--year YYYY` or `--month YYYY-MM` which will set start and end dates accordingly. GoCardless does not accept end dates that are in the future!

Transactions will be saved to the file specified in `config.ini`

# Converting transactions to CSV

Run `gocardless-to-csv convert infile.json outfile.csv`. CSV conversion is done as a separate step to avoid running afoul of 
GoCardless rate limits -- you would want to keep your `fetch` invocations to a minimum, but you can call `convert` as much as you like.

# CSV columns explained

- `bookingDate` - the date the transaction happened (you touched your card to a reader)

- `valueDate` - the date your bank saw/processed it

- `operation` - either `debit` (you got money) or `credit` (you paid money)

- `payee` - who or which organization was on the other side of the transaction

- `transactionAmount` - how much your bank charged you (in the currency of your account)

- `instructedAmount` - how much the `payee` originally charged you  (in the currency of payee's choice)

- `description` - description collected by GoCardless

- `GoCardlessRef` - unique transaction ID provided by GoCardless

# Suggested workflow

- set everything up

- configure your files to go into `%Y/%m/<account_name>.json`

- After the end of the month YYYY-MM, run `gocardless-to-csv --month YYYY-MM` and fetch all JSON files

- Convert them to CSV

- Use `hledger print` or something similar to include the CSV files into your accounting setup

# HLedger rules for importing those files

TODO