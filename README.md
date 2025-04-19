

# Constants
readonly API_MODEL="llama-3.3-70b-versatile"
readonly API_ENDPOINT="https://api.groq.com/openai/v1/chat/completions"

# Dangerous command patterns
readonly DANGEROUS_PATTERNS="rm -rf|mkfs|dd|shred|> /dev/sd|chmod 000|^:|wget|curl"

# Log error message and exit
error_exit() {
    echo "Error: $1" >&2
    exit 1
}

# Validate input query
validate_query() {
    [[ -z "$1" ]] && error_exit "No query provided. Usage: $0 'your request'"
}

# Prepare API payload
prepare_payload() {
    local query="$1"
    jq -n --arg query "$query" --arg model "$API_MODEL" '
    {
        "model": $model,
        "messages": [{
            "role": "user",
            "content": (
                "Generate exactly 5 DISTINCT Linux commands for: \($query). " +
                "Format STRICTLY as:\n" +
                "1. command1\n" +
                "2. command2\n" +
                "...\n" +
                "5. command5\n" +
                "No explanations. No markdown. No numbering beyond 5. " +
                "Never repeat similar commands."
            )
        }],
        "temperature": 0.5,
        "max_tokens": 500
    }'
}

# Fetch commands from API
fetch_commands() {
    local payload="$1"
    [[ -z "$GROQ_API_KEY" ]] && error_exit "API key not set. Export GROQ_API_KEY"

    curl -s -X POST "$API_ENDPOINT" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $GROQ_API_KEY" \
        -d "$payload"
}

# Parse and clean commands
parse_commands() {
    local raw_content="$1"
    echo "$raw_content" | jq -r '.choices[0].message.content' |
        grep -E '^[0-9]+\.' |        # Filter only numbered lines
        head -n5 |                   # Take first 5
        sed -E 's/^[0-9]+\.\s*//; s/`//g; s/\*//g' |  # Clean formatting
        awk '!seen[$0]++'            # Remove duplicates
}

# Safety check for dangerous commands
safety_check() {
    local cmd="$1"
    if [[ "$cmd" =~ $DANGEROUS_PATTERNS ]]; then
        echo "Warning: Potentially dangerous command detected!"
        read -p "Are you sure you want to continue? (y/n): " confirm
        [[ "$confirm" =~ ^[Yy]$ ]] || error_exit "Command aborted due to safety concerns."
    fi
}

# Edit command function
edit_command() {
    local cmd="$1"
    local edited_cmd

    # Allow up to 3 edits
    for ((i=1; i<=3; i++)); do
        # Prompt for editing
        read -rei "$cmd" -p "Edit command (attempt $i/3): " edited_cmd

        # If user doesn't change the command, ask if they want to continue
        if [[ "$edited_cmd" == "$cmd" ]]; then
            read -p "No changes made. Continue editing? (y/n): " continue_edit
            [[ "$continue_edit" =~ ^[Nn]$ ]] && break
        else
            # If command is changed, confirm
            read -p "Use modified command? (y/n): " confirm_edit
            if [[ "$confirm_edit" =~ ^[Yy]$ ]]; then
                cmd="$edited_cmd"
                break
            fi
        fi
    done

    echo "$cmd"
}
