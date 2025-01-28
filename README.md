# checkcert

v0003 OpenBSD

A) add domains on top
B) it loops over them and writes expiration to a file you can read from

- create the script file and make it executable
 `touch /usr/local/chheckcert.sh && chmod +x /usr/local/chheckcert.sh`
```
#!/bin/bash

# === CONFIGURATION VARIABLES ===

# Array of domains to check
DOMAINS=("google.com" "example.com" "anotherdomain.com")

# Output directory and file
OUTPUT_DIR="/usr/local/share/ssl_expiration"
OUTPUT_FILE="$OUTPUT_DIR/ssl_expiration_report.txt"
CACHE_FILE="$OUTPUT_DIR/ssl_expiration_cache.txt"
LOG_FILE="/var/log/ssl_expiration.log"

# Default permissions for the output directory and file
DIR_PERMISSIONS=700
FILE_PERMISSIONS=600

# Date format for OpenBSD's `date`
DATE_FORMAT="%b %d %H:%M:%S %Y %Z"

# In-memory cache (associative array for bash 4+)
declare -A CACHE

# === LOGGING FUNCTION ===
log() {
    echo "$(date +'%Y-%m-%d %H:%M:%S') - $*" >> "$LOG_FILE"
}

# === INITIALIZATION ===
# Ensure the output directory exists
if [ ! -d "$OUTPUT_DIR" ]; then
    mkdir -p "$OUTPUT_DIR"
    chmod "$DIR_PERMISSIONS" "$OUTPUT_DIR"
    log "Created output directory: $OUTPUT_DIR"
fi

# Load cache into memory if the file exists
if [ -f "$CACHE_FILE" ]; then
    while read -r DOMAIN CACHED_DATE; do
        CACHE["$DOMAIN"]="$CACHED_DATE"
    done < "$CACHE_FILE"
    log "Loaded cache into memory"
fi

# Initialize output buffer
OUTPUT_BUFFER=""

# Add table header to the output buffer
OUTPUT_BUFFER+=$(printf "%-25s | %-25s | %-10s\n" "Domain" "Valid Until" "Days Left")
OUTPUT_BUFFER+=$'\n'
OUTPUT_BUFFER+=$(printf "%-25s | %-25s | %-10s\n" "-------------------------" "-------------------------" "----------")
OUTPUT_BUFFER+=$'\n'

# === MAIN PROCESSING ===
# Loop through domains and fetch SSL certificate details
for DOMAIN in "${DOMAINS[@]}"; do
    log "Checking domain: $DOMAIN"

    # Check cache for existing data
    CACHED_DATE="${CACHE["$DOMAIN"]}"
    if [[ -n "$CACHED_DATE" ]]; then
        # Check if the cached date is still valid
        CACHED_TIMESTAMP=$(date -j -f "$DATE_FORMAT" "$CACHED_DATE" +%s 2>/dev/null)
        CURRENT_TIMESTAMP=$(date +%s)
        if (( CACHED_TIMESTAMP > CURRENT_TIMESTAMP )); then
            # Use cached data
            DAYS_LEFT=$(( (CACHED_TIMESTAMP - CURRENT_TIMESTAMP) / 86400 ))
            OUTPUT_BUFFER+=$(printf "%-25s | %-25s | %-10s\n" "$DOMAIN" "$CACHED_DATE" "$DAYS_LEFT")
            OUTPUT_BUFFER+=$'\n'
            log "Used cached data for $DOMAIN: $CACHED_DATE ($DAYS_LEFT days left)"
            continue
        fi
    fi

    # Fetch SSL certificate expiration date
    EXPIRATION_DATE=$(echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null | \
                       openssl x509 -noout -dates | \
                       grep 'notAfter' | \
                       awk -F= '{print $2}')
    
    if [[ -z "$EXPIRATION_DATE" ]]; then
        OUTPUT_BUFFER+=$(printf "%-25s | %-25s | %-10s\n" "$DOMAIN" "Connection Failed" "N/A")
        OUTPUT_BUFFER+=$'\n'
        log "Connection failed for $DOMAIN"
        continue
    fi

    # Convert expiration date to seconds since epoch
    EXPIRATION_TIMESTAMP=$(date -j -f "$DATE_FORMAT" "$EXPIRATION_DATE" +%s 2>/dev/null)
    if [[ -z "$EXPIRATION_TIMESTAMP" ]]; then
        OUTPUT_BUFFER+=$(printf "%-25s | %-25s | %-10s\n" "$DOMAIN" "Invalid Date Format" "N/A")
        OUTPUT_BUFFER+=$'\n'
        log "Invalid date format for $DOMAIN: $EXPIRATION_DATE"
        continue
    fi

    # Calculate days left
    CURRENT_TIMESTAMP=$(date +%s)
    DAYS_LEFT=$(( (EXPIRATION_TIMESTAMP - CURRENT_TIMESTAMP) / 86400 ))
    OUTPUT_BUFFER+=$(printf "%-25s | %-25s | %-10s\n" "$DOMAIN" "$EXPIRATION_DATE" "$DAYS_LEFT")
    OUTPUT_BUFFER+=$'\n'
    log "Fetched expiration for $DOMAIN: $EXPIRATION_DATE ($DAYS_LEFT days left)"

    # Update in-memory cache
    CACHE["$DOMAIN"]="$EXPIRATION_DATE"
done

# Write output buffer to file
echo -e "$OUTPUT_BUFFER" > "$OUTPUT_FILE"
chmod "$FILE_PERMISSIONS" "$OUTPUT_FILE"
log "SSL expiration report updated: $OUTPUT_FILE"

# Write updated cache to file
{
    for DOMAIN in "${!CACHE[@]}"; do
        echo "$DOMAIN ${CACHE["$DOMAIN"]}"
    done
} > "$CACHE_FILE"
chmod "$FILE_PERMISSIONS" "$CACHE_FILE"
log "Cache updated: $CACHE_FILE"
```
- results are stored at `/usr/local/share/ssl_expiration/ssl_expiration_report.txt`
- you can just read from that file at your .profile on login for whatever user
```
if [ -f "/usr/local/share/ssl_expiration/ssl_expiration_report.txt" ]; then
    cat "/usr/local/share/ssl_expiration/ssl_expiration_report.txt"
fi
```
- add a cronjob via `crontab -e`
- run it at any interval you like: 2 weeks at 3:30
```
30 3 */14 * * /bin/ksh /usr/local/bin/checkcert.sh
```
