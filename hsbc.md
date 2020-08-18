# HSBC - PDF to CSV

Some notes about downloading HSBC's monthly statements as PDF and converting
them into CSV files for processing. (HSBC do provide some kind of CSV statement
service, but it appears to be limited in retention and seemed to redact the
names/references for some payments).

## Fetching PDFs

HSBC offer PDF statements going back a number of years. Downloading them through
the UI is a little awkward.

I used the following script to click the download button for each statement. I
needed to repeat this for each year of statements.

:warning: Please be very careful running scripts on your online banking, as it
is a very easy way to be scammed. I would advise against running this.

```
document.querySelectorAll('.viewDownload')
    .forEach((button, idx) => {
        // The delay seems to be necessary as otherwise only one statement is
        // downloaded. Perhaps the event gets de-duped somewhere in a JS
        // framework?
        setTimeout(() => button.click(), 2000 * idx)
    })
```

## PDF to CSV

### PDF to Text

I ran the following to extract the text from the PDF:

```
gs -sDEVICE=txtwrite -o output.txt statement.pdf
```

The text was correctly labelled in the PDF (could be highlighted), so this could
just extract it. No OCR needed.

### Text to CSV

This AWK script parses the text from the statement.

This is my first time writing an AWK script and this parsing was a little
complicated. I am not sure it was really well-suited to AWK and I needed a
couple of `gawk` extensions (primarily `FIELDWIDTHS`) to get it to work.

:warning: This assumes text in the statement is in a particular format. It
assumes the payment details (name and reference) are trustworthy and don't
affect this format. It would be very easy for this to misparse data, so it
should not be relied upon.

```awk
BEGIN {
    OFS = ","
    FIELDWIDTHS = ""
}

# The PDF is converted to tabular ASCII data. Unfortunately the alignment of
# the columns is different on each page. This synchronizes FIELDWIDTHS each
# time we find the headers of the table.
#
# Fields:
# $1 = Date, e.g. 30 Jul 2020
# $2 = Payment type, e.g. VIS, CR, BP
# $3 = Comments, over multiple lines
# $4 = Paid out, e.g. 9.00
# $5 = Pain in, e.g. 9.00
# $6 = Balance, e.g. 0.00
{ _headers = 0 }
/Date +Payment type and details +Paid out +Paid in +Balance/ { _headers = 1 }
_headers == 1 {
    _fw_date = match($0, /Payment type/) - 2
    _fw_type = 3
    _fw_comment = match($0, /Paid out/) - match($0, /Payment type/) - 3
    _fw_in = match($0, /Paid in/) - match($0, /Paid out/)
    _fw_out = match($0, /Balance/) - match($0, /Paid in/)
    _fw_balance = length($0) - match($0, /Balance/) + 3
    # Note: Setting FIELDWIDTHS takes effect on the next line.
    FIELDWIDTHS = _fw_date " " _fw_type " " _fw_comment " " _fw_in " " _fw_out " " _fw_balance
}

{
    # Strip trailing/leading whitespace from each field.
    # Also strip carriage returns which are mixed in.
    gsub(/(^ +| +$|\r)/, "", $1)
    gsub(/(^ +| +$|\r)/, "", $2)
    gsub(/(^ +| +$|\r)/, "", $3)
    gsub(/(^ +| +$|\r)/, "", $4)
    gsub(/(^ +| +$|\r)/, "", $5)
    gsub(/(^ +| +$|\r)/, "", $6)
    # The numeric fields use , but this clashes with our CSV terminator.
    gsub(",", "", $4);
    gsub(",", "", $5);
    gsub(",", "", $6);
}

function parse_month(month) {
    switch (month) {
        case "Jan": return "01"
        case "Feb": return "02"
        case "Mar": return "03"
        case "Apr": return "04"
        case "May": return "05"
        case "Jun": return "06"
        case "Jul": return "07"
        case "Aug": return "08"
        case "Sep": return "09"
        case "Oct": return "10"
        case "Nov": return "11"
        case "Dec": return "12"
    }
}

# Match date in $1 and remember it. There can be multiple transactions for
# each date.
match($1, /([0-9]+) ([a-zA-Z]{3}) ([0-9]+)/, groups) {
    day = groups[1]
    month = parse_month(groups[2])
    year = "20" groups[3]
}

# Parse lines between BALANCEBROUGHTFORWARD and BALANCECARRIEDFORWARD.
$3 ~ /BALANCEBROUGHTFORWARD/ { _parsing = 1; next }
$3 ~ /BALANCECARRIEDFORWARD/ { _parsing = 0 }

function print_row() {
    print tx_date, tx_type, tx_comment, tx_out, tx_in, tx_balance
}

# When there is a payment type, it is the start of a transaction.
# If we're parsing a transaction, output a CSV row.
$2 != "" && _in_tx == 1 { print_row() }
_parsing == 1 && $2 != "" {
    _in_tx = 1
    tx_date = year "-" month "-" day
    tx_type = $2
    tx_comment = ""
    tx_out = ""
    tx_in = ""
    tx_balance = ""
}
# Parsing a transaction terminates either at the next transaction or when we
# stop parsing altogether.
_in_tx == 1 && _parsing == 0 { print_row(); _in_tx = 0 }
# Parse each line of transaction, which may be over multiple lines.
# Each field will only be on one of the transaction's lines (so we don't care
# about overwriting values), except comment which we concat with |.
_in_tx == 1 && $3 != "" {
    if (length(tx_comment) == 0)
        tx_comment = $3
    else
        tx_comment = tx_comment " " $3
}
_in_tx == 1 && $4 != "" { tx_out = $4 }
_in_tx == 1 && $5 != "" { tx_in = $5 }
_in_tx == 1 && $6 != "" { tx_balance = $6 }
```

