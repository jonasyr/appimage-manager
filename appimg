#!/bin/bash

# handle_appimage.sh
# A comprehensive script to manage AppImage files:
# - Add new AppImages
# - Edit existing AppImages
# - Update AppImage versions
# - Delete AppImages
# - Automatically handles moving, setting executable permissions, and creating/updating .desktop files

set -e

# Constants
DEFAULT_DEST_DIR="$HOME/Applications"
ICON_DIR="$HOME/Icons"
DESKTOP_DIR="$HOME/.local/share/applications"
CONFIG_DIR="$HOME/.config/handle_appimage"
DATABASE_FILE="$CONFIG_DIR/appimages.csv"

# Ensure necessary directories exist
mkdir -p "$DEFAULT_DEST_DIR"
mkdir -p "$ICON_DIR"
mkdir -p "$DESKTOP_DIR"
mkdir -p "$CONFIG_DIR"

# Function to print verbose messages
verbose() {
    echo -e "[INFO] $1"
}

# Function to print error messages and exit
error_exit() {
    echo -e "[ERROR] $1" >&2
    exit 1
}

# Function to prompt for input with a default value
prompt_with_default() {
    local PROMPT_MESSAGE=$1
    local DEFAULT_VALUE=$2
    read -p "$PROMPT_MESSAGE [$DEFAULT_VALUE]: " INPUT
    if [ -z "$INPUT" ]; then
        INPUT=$DEFAULT_VALUE
    fi
    echo "$INPUT"
}

# Function to expand tilde and resolve absolute paths
expand_path() {
    local INPUT_PATH="$1"
    # Replace ~ with $HOME
    if [[ "$INPUT_PATH" == ~* ]]; then
        INPUT_PATH="${INPUT_PATH/#\~/$HOME}"
    fi
    # Resolve to absolute path
    REAL_PATH=$(realpath "$INPUT_PATH" 2>/dev/null || echo "$INPUT_PATH")
    echo "$REAL_PATH"
}

find_recent_file() {
    local EXTENSIONS=("$@")
    local FIND_EXPR=()
    for EXT in "${EXTENSIONS[@]}"; do
        FIND_EXPR+=("-iname" "*.$EXT" "-o")
    done
    # Remove the trailing '-o'
    unset 'FIND_EXPR[${#FIND_EXPR[@]}-1]'
    local RECENT_FILE=$(find "$HOME/Downloads" -maxdepth 1 -type f \( "${FIND_EXPR[@]}" \) -printf '%T@ %p\n' 2>/dev/null | sort -n | tail -n1 | awk '{print $2}')
    echo "$RECENT_FILE"
}

# Function to initialize the database by scanning common directories
initialize_database() {
    verbose "Initializing AppImage database by scanning common directories..."
    > "$DATABASE_FILE"  # Empty the database file

    # Scan the Applications directory for AppImages
    for app in "$DEFAULT_DEST_DIR"/*.AppImage; do
        if [ -f "$app" ]; then
            APPIMAGE_PATH=$(realpath "$app")
            APP_NAME=$(basename "$app" .AppImage)
            # Find corresponding .desktop file
            DESKTOP_FILE_PATH="$DESKTOP_DIR/${APP_NAME}.desktop"
            if [ -f "$DESKTOP_FILE_PATH" ]; then
                ICON_PATH=$(grep '^Icon=' "$DESKTOP_FILE_PATH" | cut -d'=' -f2-)
            else
                ICON_PATH=""
                DESKTOP_FILE_PATH=""
            fi
            echo "$APP_NAME|$APPIMAGE_PATH|$DESKTOP_FILE_PATH|$ICON_PATH" >> "$DATABASE_FILE"
        fi
    done

    verbose "Database initialization completed."
}


# Function to load the database into arrays
load_database() {
    declare -gA APPIMAGES
    declare -ga APP_NAMES
    APPIMAGES=()
    APP_NAMES=()
    if [ -f "$DATABASE_FILE" ]; then
        while IFS='|' read -r APP_NAME APPIMAGE_PATH DESKTOP_FILE_PATH ICON_PATH; do
            APPIMAGES["$APP_NAME"]="$APPIMAGE_PATH|$DESKTOP_FILE_PATH|$ICON_PATH"
            APP_NAMES+=("$APP_NAME")
        done < "$DATABASE_FILE"
    fi
}

# Function to save an entry to the database
save_to_database() {
    local APP_NAME="$1"
    local APPIMAGE_PATH="$2"
    local DESKTOP_FILE_PATH="$3"
    local ICON_PATH="$4"
    echo "$APP_NAME|$APPIMAGE_PATH|$DESKTOP_FILE_PATH|$ICON_PATH" >> "$DATABASE_FILE"
}

# Function to update an entry in the database
update_database_entry() {
    local APP_NAME="$1"
    local NEW_APPIMAGE_PATH="$2"
    local NEW_DESKTOP_FILE_PATH="$3"
    local NEW_ICON_PATH="$4"
    verbose "Updating database entry for '$APP_NAME'."

    # Use temporary file for updating
    tmp_file=$(mktemp) || error_exit "Failed to create temporary file in update_database_entry."
    # touch "$tmp_file" || error_exit "Failed to initialize temporary file."

    entry_found=false
    while IFS='|' read -r NAME APP_PATH DESKTOP ICON; do
        if [ "$NAME" == "$APP_NAME" ]; then
            echo "$APP_NAME|$NEW_APPIMAGE_PATH|$NEW_DESKTOP_FILE_PATH|$NEW_ICON_PATH" >> "$tmp_file"
            entry_found=true
            verbose "Entry '$APP_NAME' found and updated."
        elif [ -n "$NAME" ]; then
            echo "$NAME|$APP_PATH|$DESKTOP|$ICON" >> "$tmp_file"
        fi
    done < "$DATABASE_FILE"

    if [ "$entry_found" = false ]; then
        error_exit "Entry '$APP_NAME' not found in the database."
    fi

    mv "$tmp_file" "$DATABASE_FILE" || error_exit "Failed to update database file in update_database_entry."
    verbose "Database file updated successfully."
}

# Function to remove an entry from the database
remove_from_database() {
    local APP_NAME="$1"
    verbose "Removing '$APP_NAME' from the database."

    # Use mktemp without a directory path to ensure it creates a file
    tmp_file=$(mktemp) || error_exit "Failed to create temporary file in remove_from_database."

    entry_found=false
    while IFS='|' read -r NAME APP_PATH DESKTOP ICON; do
        if [ "$NAME" != "$APP_NAME" ] && [ -n "$NAME" ]; then
            echo "$NAME|$APP_PATH|$DESKTOP|$ICON" >> "$tmp_file"
        else
            entry_found=true
            verbose "Entry '$APP_NAME' found and will be removed."
        fi
    done < "$DATABASE_FILE"

    if [ "$entry_found" = false ]; then
        error_exit "Entry '$APP_NAME' not found in the database."
    fi

    verbose "tmp_file is '$tmp_file'"
    verbose "DATABASE_FILE is '$DATABASE_FILE'"
    ls -l "$tmp_file" "$DATABASE_FILE" || verbose "One of the files does not exist."
    cat "$tmp_file"

    mv "$tmp_file" "$DATABASE_FILE" || error_exit "Failed to update database file in remove_from_database."
    verbose "Database file updated successfully."
}


# Function to list managed AppImages
list_appimages() {
    if [ ! -f "$DATABASE_FILE" ] || [ ! -s "$DATABASE_FILE" ]; then
        echo "No AppImages found in the database."
        return 1
    fi
    echo "Managed AppImages:"
    local INDEX=1
    for APP_NAME in "${APP_NAMES[@]}"; do
        echo "  [$INDEX] $APP_NAME"
        INDEX=$((INDEX + 1))
    done
    return 0
}

# Function to select an AppImage from the list
select_appimage() {
    list_appimages
    if [ $? -ne 0 ]; then
        return 1
    fi

    local APP_COUNT=${#APP_NAMES[@]}
    local CHOICE
    while true; do
        echo -n "Enter the number of the AppImage: " >&2
        read CHOICE
        if [[ "$CHOICE" =~ ^[0-9]+$ ]] && [ "$CHOICE" -ge 1 ] && [ "$CHOICE" -le "$APP_COUNT" ]; then
            APP_NAME="${APP_NAMES[$((CHOICE - 1))]}"
            SELECTED_APP_NAME="$APP_NAME"
            SELECTED_APP_INFO="${APPIMAGES[$APP_NAME]}"
            return 0
        else
            echo "Invalid selection. Please enter a number between 1 and $APP_COUNT." >&2
        fi
    done
}

# Function to create or update a .desktop file
create_desktop_file() {
    local APP_NAME="$1"
    local EXEC_CMD="$2"
    local ICON_PATH="$3"
    local COMMENT="$4"
    local DESKTOP_FILE_PATH="$DESKTOP_DIR/${APP_NAME}.desktop"

    # Check if .desktop file already exists
    if [ -f "$DESKTOP_FILE_PATH" ]; then
        read -p "A .desktop file named '${APP_NAME}.desktop' already exists. Overwrite? (y/N): " OVERWRITE
        case "$OVERWRITE" in
            y|Y )
                verbose "Overwriting existing .desktop file."
                ;;
            * )
                error_exit "Aborting to prevent overwriting the existing .desktop file."
                ;;
        esac
    fi

    verbose "Creating .desktop file at: $DESKTOP_FILE_PATH"

    # Write the .desktop file content
    cat > "$DESKTOP_FILE_PATH" <<EOL
[Desktop Entry]
Name=$APP_NAME
Exec="$EXEC_CMD" %U
Icon=$ICON_PATH
Type=Application
Categories=Utility;
Terminal=false
Comment=$COMMENT
EOL

    # Make the .desktop file executable
    chmod +x "$DESKTOP_FILE_PATH" || error_exit "Failed to make the .desktop file executable."

    verbose "Successfully created the .desktop file."
}

# Function to update the desktop database
update_desktop_db() {
    if command -v update-desktop-database >/dev/null 2>&1; then
        verbose "Updating desktop database."
        update-desktop-database "$DESKTOP_DIR" || verbose "Failed to update desktop database."
    fi
}

# Function to add a new AppImage
add_appimage() {
    echo "=== Add a New AppImage ==="

    # Find the most recent .AppImage in Downloads
    RECENT_APPIMAGE=$(find_recent_file "AppImage" "appimage")
    if [ -n "$RECENT_APPIMAGE" ]; then
        DEFAULT_APPIMAGE=$(basename "$RECENT_APPIMAGE")
        verbose "Detected recent AppImage in Downloads: $DEFAULT_APPIMAGE"
    else
        DEFAULT_APPIMAGE=""
    fi

    # Prompt for AppImage path with default
    if [ -n "$DEFAULT_APPIMAGE" ]; then
        PROMPT_APPIMAGE="Enter the path to the AppImage file (default: $DEFAULT_APPIMAGE): "
    else
        PROMPT_APPIMAGE="Enter the path to the AppImage file: "
    fi
    read -p "$PROMPT_APPIMAGE" APPIMAGE_PATH_INPUT
    if [ -z "$APPIMAGE_PATH_INPUT" ] && [ -n "$DEFAULT_APPIMAGE" ]; then
        APPIMAGE_PATH_INPUT="$HOME/Downloads/$DEFAULT_APPIMAGE"
    fi
    APPIMAGE_PATH=$(expand_path "$APPIMAGE_PATH_INPUT")

    # Validate AppImage file
    if [ ! -f "$APPIMAGE_PATH" ]; then
        error_exit "The specified AppImage file does not exist: $APPIMAGE_PATH"
    fi

    # Prompt for destination directory with default
    DEST_DIR_INPUT=$(prompt_with_default "Enter the destination directory for AppImages" "$DEFAULT_DEST_DIR")
    DEST_DIR=$(expand_path "$DEST_DIR_INPUT")

    # Create destination directory if it doesn't exist
    if [ ! -d "$DEST_DIR" ]; then
        verbose "Destination directory does not exist. Creating: $DEST_DIR"
        mkdir -p "$DEST_DIR" || error_exit "Failed to create destination directory: $DEST_DIR"
    else
        verbose "Destination directory exists: $DEST_DIR"
    fi

    # Find the most recent icon file in Downloads
    RECENT_ICON=$(find_recent_file "png" "svg" "ico" "jpg" "jpeg")
    if [ -n "$RECENT_ICON" ]; then
        DEFAULT_ICON=$(basename "$RECENT_ICON")
        verbose "Detected recent icon in Downloads: $DEFAULT_ICON"
    else
        DEFAULT_ICON=""
    fi

    # Prompt for icon path with default
    if [ -n "$DEFAULT_ICON" ]; then
        PROMPT_ICON="Enter the path to the icon file (default: $DEFAULT_ICON): "
    else
        PROMPT_ICON="Enter the path to the icon file (leave blank to extract from AppImage if possible): "
    fi
    read -p "$PROMPT_ICON" ICON_PATH_INPUT
    if [ -z "$ICON_PATH_INPUT" ] && [ -n "$DEFAULT_ICON" ]; then
        ICON_PATH_INPUT="$HOME/Downloads/$DEFAULT_ICON"
    fi

    if [ -n "$ICON_PATH_INPUT" ]; then
        ICON_PATH=$(expand_path "$ICON_PATH_INPUT")
        if [ ! -f "$ICON_PATH" ]; then
            error_exit "The specified icon file does not exist: $ICON_PATH"
        else
            verbose "Icon file provided: $ICON_PATH"
            # Move the icon to ICON_DIR
            ICON_FILENAME=$(basename "$ICON_PATH")
            DEST_ICON_PATH="$ICON_DIR/$ICON_FILENAME"
            if [ "$ICON_PATH" != "$DEST_ICON_PATH" ]; then
                verbose "Moving icon to: $DEST_ICON_PATH"
                mv "$ICON_PATH" "$DEST_ICON_PATH" || error_exit "Failed to move icon file."
                ICON_PATH="$DEST_ICON_PATH"
            else
                verbose "Icon is already in the icons directory."
            fi
        fi
    else
        verbose "No icon file provided. Attempting to extract from AppImage."
        # Attempt to extract icon using appimagetool or similar if possible
        # For simplicity, we'll skip extraction and proceed without an icon
        ICON_PATH=""
        verbose "No icon will be set."
    fi

    # Determine the application name
    # Attempt to extract the name from the AppImage metadata
    # This requires the AppImage to support --appimage-info
    if "$APPIMAGE_PATH" --appimage-info >/dev/null 2>&1; then
        APP_INFO=$("$APPIMAGE_PATH" --appimage-info 2>/dev/null || true)
        if [ -n "$APP_INFO" ]; then
            APP_NAME=$(echo "$APP_INFO" | grep '^name=' | head -n1 | cut -d'=' -f2-)
        else
            APP_NAME=""
        fi
    else
        APP_NAME=""
    fi

    if [ -z "$APP_NAME" ]; then
        # Fallback: use the filename without extension
        APP_NAME=$(basename "$APPIMAGE_PATH" .AppImage)
        verbose "Could not extract application name. Using filename without extension: $APP_NAME"
    else
        verbose "Extracted application name: $APP_NAME"
    fi

    # Prompt for application name if still not determined or user wants to change
    CURRENT_APP_NAME="$APP_NAME"
    read -p "Enter the application name (leave blank to use '$CURRENT_APP_NAME'): " USER_APP_NAME
    if [ -n "$USER_APP_NAME" ]; then
        APP_NAME="$USER_APP_NAME"
    elif [ -z "$APP_NAME" ]; then
        APP_NAME=$(basename "$APPIMAGE_PATH" .AppImage)
        verbose "Using filename without extension as application name: $APP_NAME"
    fi

    # Move the AppImage to the destination directory and rename it
    APPIMAGE_NAME="${APP_NAME}.AppImage"
    DEST_APPIMAGE_PATH="$DEST_DIR/$APPIMAGE_NAME"

    verbose "Moving AppImage to destination directory: $DEST_APPIMAGE_PATH"
    mv "$APPIMAGE_PATH" "$DEST_APPIMAGE_PATH" || error_exit "Failed to move AppImage to destination."

    # Make the AppImage executable
    verbose "Making the AppImage executable."
    chmod +x "$DEST_APPIMAGE_PATH" || error_exit "Failed to make the AppImage executable."

    # Determine the exec command
    EXEC_CMD="$DEST_APPIMAGE_PATH"

    # Determine the icon path
    if [ -z "$ICON_PATH" ]; then
        # Optionally, implement extraction of icon from AppImage here
        ICON_PATH=""
        verbose "No icon will be set for the .desktop file."
    fi

    # Create the .desktop file
    DESKTOP_FILE_PATH="$DESKTOP_DIR/${APP_NAME}.desktop"
    create_desktop_file "$APP_NAME" "$EXEC_CMD" "$ICON_PATH" "AppImage application: $APP_NAME"

    # Save to database
    save_to_database "$APP_NAME" "$EXEC_CMD" "$DESKTOP_FILE_PATH" "$ICON_PATH"

    # Update desktop database
    update_desktop_db

    verbose "AppImage handling completed successfully."
}

# Function to edit an existing AppImage
edit_appimage() {
    echo "=== Edit an Existing AppImage ==="

    # Select an AppImage
    select_appimage || error_exit "No AppImages available to edit."
    IFS='|' read -r SELECTED_APPIMAGE_PATH SELECTED_DESKTOP_FILE_PATH SELECTED_ICON_PATH <<< "$SELECTED_APP_INFO"

    verbose "Selected AppImage: $SELECTED_APP_NAME"
    verbose "SELECTED_APPIMAGE_PATH: $SELECTED_APPIMAGE_PATH"
    verbose "SELECTED_DESKTOP_FILE_PATH: $SELECTED_DESKTOP_FILE_PATH"
    verbose "SELECTED_ICON_PATH: $SELECTED_ICON_PATH"

    # Store the old application name before any changes
    OLD_APP_NAME="$SELECTED_APP_NAME"

    # Check if the selected AppImage still exists
    if [ ! -f "$SELECTED_APPIMAGE_PATH" ]; then
        error_exit "The selected AppImage file does not exist: $SELECTED_APPIMAGE_PATH"
    fi

    # Determine the corresponding .desktop file
    if [ -f "$SELECTED_DESKTOP_FILE_PATH" ]; then
        verbose "Corresponding .desktop file found: $SELECTED_DESKTOP_FILE_PATH"
    else
        verbose "No corresponding .desktop file found for '$SELECTED_APP_NAME'."
    fi

    # Present edit options
    echo "What would you like to edit?"
    echo "  [1] Application Name"
    echo "  [2] Icon"
    echo "  [3] Executable Path"
    echo "  [4] All of the above"
    echo "  [5] Cancel"

    read -p "Enter your choice [1-5]: " EDIT_CHOICE

    # Initialize variables to track changes
    NEW_APP_NAME=""
    NEW_ICON_PATH=""
    NEW_EXEC_CMD=""

    case "$EDIT_CHOICE" in
        1)
            # Edit Application Name
            read -p "Enter the new application name (leave blank to keep current: '$SELECTED_APP_NAME'): " NEW_APP_NAME
            ;;
        2)
            # Edit Icon
            # Find the most recent icon file in Downloads
            RECENT_ICON=$(find_recent_file "png" "svg" "ico" "jpg" "jpeg")
            if [ -n "$RECENT_ICON" ]; then
                DEFAULT_ICON=$(basename "$RECENT_ICON")
                verbose "Detected recent icon in Downloads: $DEFAULT_ICON"
            else
                DEFAULT_ICON=""
            fi

            if [ -n "$DEFAULT_ICON" ]; then
                PROMPT_ICON="Enter the path to the new icon file (default: $DEFAULT_ICON, leave blank to remove icon): "
            else
                PROMPT_ICON="Enter the path to the new icon file (leave blank to remove icon): "
            fi
            read -p "$PROMPT_ICON" NEW_ICON_PATH_INPUT
            if [ -z "$NEW_ICON_PATH_INPUT" ] && [ -n "$DEFAULT_ICON" ]; then
                NEW_ICON_PATH_INPUT="$HOME/Downloads/$DEFAULT_ICON"
            fi

            if [ -n "$NEW_ICON_PATH_INPUT" ]; then
                NEW_ICON_PATH=$(expand_path "$NEW_ICON_PATH_INPUT")
                if [ ! -f "$NEW_ICON_PATH" ]; then
                    error_exit "The specified icon file does not exist: $NEW_ICON_PATH"
                else
                    verbose "New icon file provided: $NEW_ICON_PATH"
                    # Move the icon to ICON_DIR
                    ICON_FILENAME=$(basename "$NEW_ICON_PATH")
                    DEST_ICON_PATH="$ICON_DIR/$ICON_FILENAME"
                    if [ "$NEW_ICON_PATH" != "$DEST_ICON_PATH" ]; then
                        verbose "Moving new icon to: $DEST_ICON_PATH"
                        mv "$NEW_ICON_PATH" "$DEST_ICON_PATH" || error_exit "Failed to move icon file."
                        NEW_ICON_PATH="$DEST_ICON_PATH"
                    else
                        verbose "Icon is already in the icons directory."
                    fi
                fi
            else
                verbose "No icon file provided. Removing icon from .desktop file."
                NEW_ICON_PATH=""
            fi
            ;;
        3)
            # Edit Executable Path
            # Find the most recent .AppImage in Downloads
            RECENT_APPIMAGE=$(find_recent_file "AppImage" "appimage")
            if [ -n "$RECENT_APPIMAGE" ]; then
                DEFAULT_APPIMAGE=$(basename "$RECENT_APPIMAGE")
                verbose "Detected recent AppImage in Downloads: $DEFAULT_APPIMAGE"
            else
                DEFAULT_APPIMAGE=""
            fi

            if [ -n "$DEFAULT_APPIMAGE" ]; then
                PROMPT_EXEC="Enter the path to the new AppImage file (default: $DEFAULT_APPIMAGE): "
            else
                PROMPT_EXEC="Enter the path to the new AppImage file: "
            fi
            read -p "$PROMPT_EXEC" NEW_EXEC_PATH_INPUT
            if [ -z "$NEW_EXEC_PATH_INPUT" ] && [ -n "$DEFAULT_APPIMAGE" ]; then
                NEW_EXEC_PATH_INPUT="$HOME/Downloads/$DEFAULT_APPIMAGE"
            fi
            if [ -n "$NEW_EXEC_PATH_INPUT" ]; then
                NEW_EXEC_CMD=$(expand_path "$NEW_EXEC_PATH_INPUT")
                if [ ! -f "$NEW_EXEC_CMD" ]; then
                    error_exit "The specified AppImage file does not exist: $NEW_EXEC_CMD"
                else
                    verbose "New AppImage path provided: $NEW_EXEC_CMD"
                    # Move the new AppImage to the destination directory
                    APPIMAGE_NAME_NEW="${SELECTED_APP_NAME}.AppImage"
                    DEST_NEW_EXEC_PATH="$DEFAULT_DEST_DIR/$APPIMAGE_NAME_NEW"
                    verbose "Moving new AppImage to destination directory: $DEST_NEW_EXEC_PATH"
                    mv "$NEW_EXEC_CMD" "$DEST_NEW_EXEC_PATH" || error_exit "Failed to move AppImage to destination."
                    NEW_EXEC_CMD="$DEST_NEW_EXEC_PATH"
                    # Make the new AppImage executable
                    verbose "Making the new AppImage executable."
                    chmod +x "$NEW_EXEC_CMD" || error_exit "Failed to make the new AppImage executable."
                fi
            else
                verbose "Executable path remains unchanged."
            fi
            ;;
        4)
            # Edit All of the Above
            # Application Name
            read -p "Enter the new application name (leave blank to keep current: '$SELECTED_APP_NAME'): " NEW_APP_NAME

            # Icon Path
            # Find the most recent icon file in Downloads
            RECENT_ICON=$(find_recent_file "png" "svg" "ico" "jpg" "jpeg")
            if [ -n "$RECENT_ICON" ]; then
                DEFAULT_ICON=$(basename "$RECENT_ICON")
                verbose "Detected recent icon in Downloads: $DEFAULT_ICON"
            else
                DEFAULT_ICON=""
            fi

            if [ -n "$DEFAULT_ICON" ]; then
                PROMPT_ICON="Enter the path to the new icon file (default: $DEFAULT_ICON, leave blank to remove icon): "
            else
                PROMPT_ICON="Enter the path to the new icon file (leave blank to remove icon): "
            fi
            read -p "$PROMPT_ICON" NEW_ICON_PATH_INPUT
            if [ -z "$NEW_ICON_PATH_INPUT" ] && [ -n "$DEFAULT_ICON" ]; then
                NEW_ICON_PATH_INPUT="$HOME/Downloads/$DEFAULT_ICON"
            fi

            if [ -n "$NEW_ICON_PATH_INPUT" ]; then
                NEW_ICON_PATH=$(expand_path "$NEW_ICON_PATH_INPUT")
                if [ ! -f "$NEW_ICON_PATH" ]; then
                    error_exit "The specified icon file does not exist: $NEW_ICON_PATH"
                else
                    verbose "New icon file provided: $NEW_ICON_PATH"
                    # Move the icon to ICON_DIR
                    ICON_FILENAME=$(basename "$NEW_ICON_PATH")
                    DEST_ICON_PATH="$ICON_DIR/$ICON_FILENAME"
                    if [ "$NEW_ICON_PATH" != "$DEST_ICON_PATH" ]; then
                        verbose "Moving new icon to: $DEST_ICON_PATH"
                        mv "$NEW_ICON_PATH" "$DEST_ICON_PATH" || error_exit "Failed to move icon file."
                        NEW_ICON_PATH="$DEST_ICON_PATH"
                    else
                        verbose "Icon is already in the icons directory."
                    fi
                fi
            else
                verbose "No icon file provided. Removing icon from .desktop file."
                NEW_ICON_PATH=""
            fi

            # Executable Path
            # Find the most recent .AppImage in Downloads
            RECENT_APPIMAGE=$(find_recent_file "AppImage" "appimage")
            if [ -n "$RECENT_APPIMAGE" ]; then
                DEFAULT_APPIMAGE=$(basename "$RECENT_APPIMAGE")
                verbose "Detected recent AppImage in Downloads: $DEFAULT_APPIMAGE"
            else
                DEFAULT_APPIMAGE=""
            fi

            if [ -n "$DEFAULT_APPIMAGE" ]; then
                PROMPT_EXEC="Enter the path to the new AppImage file (default: $DEFAULT_APPIMAGE): "
            else
                PROMPT_EXEC="Enter the path to the new AppImage file: "
            fi
            read -p "$PROMPT_EXEC" NEW_EXEC_PATH_INPUT
            if [ -z "$NEW_EXEC_PATH_INPUT" ] && [ -n "$DEFAULT_APPIMAGE" ]; then
                NEW_EXEC_PATH_INPUT="$HOME/Downloads/$DEFAULT_APPIMAGE"
            fi
            if [ -n "$NEW_EXEC_PATH_INPUT" ]; then
                NEW_EXEC_CMD=$(expand_path "$NEW_EXEC_PATH_INPUT")
                if [ ! -f "$NEW_EXEC_CMD" ]; then
                    error_exit "The specified AppImage file does not exist: $NEW_EXEC_CMD"
                else
                    verbose "New AppImage path provided: $NEW_EXEC_CMD"
                    # Move the new AppImage to the destination directory
                    APPIMAGE_NAME_NEW="${NEW_APP_NAME:-$SELECTED_APP_NAME}.AppImage"
                    DEST_NEW_EXEC_PATH="$DEFAULT_DEST_DIR/$APPIMAGE_NAME_NEW"
                    verbose "Moving new AppImage to destination directory: $DEST_NEW_EXEC_PATH"
                    mv "$NEW_EXEC_CMD" "$DEST_NEW_EXEC_PATH" || error_exit "Failed to move AppImage to destination."
                    NEW_EXEC_CMD="$DEST_NEW_EXEC_PATH"
                    # Make the new AppImage executable
                    verbose "Making the new AppImage executable."
                    chmod +x "$NEW_EXEC_CMD" || error_exit "Failed to make the new AppImage executable."
                fi
            else
                verbose "Executable path remains unchanged."
            fi
            ;;
        5)
            echo "Edit operation canceled."
            return 0
            ;;
        *)
            echo "Invalid choice. Aborting edit operation."
            return 1
            ;;
    esac

    # Update the .desktop file
    if [ -f "$SELECTED_DESKTOP_FILE_PATH" ]; then
        # Update Name if changed
        if [ -n "$NEW_APP_NAME" ]; then
            # Rename the .desktop file
            NEW_DESKTOP_FILE_NAME="${NEW_APP_NAME}.desktop"
            NEW_DESKTOP_FILE_PATH="$DESKTOP_DIR/$NEW_DESKTOP_FILE_NAME"
            mv "$SELECTED_DESKTOP_FILE_PATH" "$NEW_DESKTOP_FILE_PATH" || error_exit "Failed to rename .desktop file."
            verbose "Application name updated to '$NEW_APP_NAME'. Renamed .desktop file."
        else
            NEW_DESKTOP_FILE_PATH="$SELECTED_DESKTOP_FILE_PATH"
            NEW_APP_NAME="$SELECTED_APP_NAME"
        fi

        # Update Exec if changed
        if [ -n "$NEW_EXEC_CMD" ]; then
            sed -i "s|^Exec=.*|Exec=\"$NEW_EXEC_CMD\" %U|" "$NEW_DESKTOP_FILE_PATH"
            verbose "Updated Exec command in .desktop file."
        fi

        # Update Icon if changed
        if [ "$EDIT_CHOICE" -eq 2 ] || [ "$EDIT_CHOICE" -eq 4 ]; then
            if [ -n "$NEW_ICON_PATH" ]; then
                sed -i "s|^Icon=.*|Icon=$NEW_ICON_PATH|" "$NEW_DESKTOP_FILE_PATH"
                verbose "Updated Icon in .desktop file."
            else
                # Remove the Icon line
                sed -i "/^Icon=/d" "$NEW_DESKTOP_FILE_PATH"
                verbose "Removed Icon from .desktop file."
            fi
        fi

        # Update Name in .desktop file if changed
        if [ -n "$NEW_APP_NAME" ]; then
            sed -i "s|^Name=.*|Name=$NEW_APP_NAME|" "$NEW_DESKTOP_FILE_PATH"
            verbose "Updated Name in .desktop file."
        fi

    else
        verbose "No .desktop file to update."
    fi

    # Determine the new paths
    if [ -n "$NEW_APP_NAME" ]; then
        APP_NAME_TO_SAVE="$NEW_APP_NAME"
    else
        APP_NAME_TO_SAVE="$SELECTED_APP_NAME"
    fi

    if [ -n "$NEW_EXEC_CMD" ]; then
        EXEC_TO_SAVE="$NEW_EXEC_CMD"
    else
        EXEC_TO_SAVE="$SELECTED_APPIMAGE_PATH"
    fi

    if [ -n "$NEW_DESKTOP_FILE_PATH" ]; then
        DESKTOP_TO_SAVE="$NEW_DESKTOP_FILE_PATH"
    else
        DESKTOP_TO_SAVE="$SELECTED_DESKTOP_FILE_PATH"
    fi

    if [ -n "$NEW_ICON_PATH" ]; then
        ICON_TO_SAVE="$NEW_ICON_PATH"
    else
        ICON_TO_SAVE="$SELECTED_ICON_PATH"
    fi

    # If the application name has changed, rename AppImage and .desktop file, and update database
    if [ "$APP_NAME_TO_SAVE" != "$OLD_APP_NAME" ]; then
        # Rename AppImage file
        OLD_APPIMAGE_PATH="$SELECTED_APPIMAGE_PATH"
        NEW_APPIMAGE_PATH="$DEFAULT_DEST_DIR/${APP_NAME_TO_SAVE}.AppImage"
        if [ ! -f "$OLD_APPIMAGE_PATH" ]; then
            error_exit "AppImage file '$OLD_APPIMAGE_PATH' does not exist. Cannot rename."
        fi
        mv "$OLD_APPIMAGE_PATH" "$NEW_APPIMAGE_PATH" || error_exit "Failed to rename AppImage file."
        EXEC_TO_SAVE="$NEW_APPIMAGE_PATH"
        verbose "Renamed AppImage file to '$NEW_APPIMAGE_PATH'."
        # Update Exec path in .desktop file
        sed -i "s|^Exec=.*|Exec=\"$EXEC_TO_SAVE\" %U|" "$DESKTOP_TO_SAVE"
        verbose "Updated Exec path in .desktop file."
        # Remove old database entry and add the new one
        remove_from_database "$OLD_APP_NAME"

        # **Add the new entry to the database**
        save_to_database "$APP_NAME_TO_SAVE" "$EXEC_TO_SAVE" "$DESKTOP_TO_SAVE" "$ICON_TO_SAVE"
    else
        # If app name hasn't changed, just update the database entry
        update_database_entry "$APP_NAME_TO_SAVE" "$EXEC_TO_SAVE" "$DESKTOP_TO_SAVE" "$ICON_TO_SAVE"
    fi

    # Update desktop database
    update_desktop_db

    verbose "Edit operation completed successfully."
}



# Function to update an AppImage version
update_appimage() {
    echo "=== Update an AppImage Version ==="

    # Select an AppImage
    select_appimage || error_exit "No AppImages available to update."
    IFS='|' read -r SELECTED_APPIMAGE_PATH SELECTED_DESKTOP_FILE_PATH SELECTED_ICON_PATH <<< "$SELECTED_APP_INFO"

    verbose "Selected AppImage: $SELECTED_APP_NAME"

    # Check if the selected AppImage still exists
    if [ ! -f "$SELECTED_APPIMAGE_PATH" ]; then
        error_exit "The selected AppImage file does not exist: $SELECTED_APPIMAGE_PATH"
    fi

    # Prompt for new AppImage path
    read -p "Enter the path to the new version of the AppImage: " NEW_APPIMAGE_PATH_INPUT
    NEW_APPIMAGE_PATH=$(expand_path "$NEW_APPIMAGE_PATH_INPUT")

    # Validate new AppImage file
    if [ ! -f "$NEW_APPIMAGE_PATH" ]; then
        error_exit "The specified new AppImage file does not exist: $NEW_APPIMAGE_PATH"
    fi

    # Confirm replacement
    read -p "Are you sure you want to replace '$SELECTED_APP_NAME' with '$(basename "$NEW_APPIMAGE_PATH")'? (y/N): " CONFIRM
    case "$CONFIRM" in
        y|Y )
            ;;
        * )
            echo "Update operation canceled."
            return 0
            ;;
    esac

    # Backup the old AppImage
    BACKUP_PATH="${SELECTED_APPIMAGE_PATH}.backup_$(date +%Y%m%d%H%M%S)"
    verbose "Backing up the existing AppImage to: $BACKUP_PATH"
    mv "$SELECTED_APPIMAGE_PATH" "$BACKUP_PATH" || error_exit "Failed to backup the existing AppImage."

    # Move the new AppImage to the destination directory
    APPIMAGE_NAME_NEW=$(basename "$NEW_APPIMAGE_PATH")
    DEST_APPIMAGE_PATH="$DEFAULT_DEST_DIR/$APPIMAGE_NAME_NEW"

    verbose "Moving new AppImage to destination directory: $DEST_APPIMAGE_PATH"
    mv "$NEW_APPIMAGE_PATH" "$DEST_APPIMAGE_PATH" || {
        # Restore backup if move fails
        mv "$BACKUP_PATH" "$SELECTED_APPIMAGE_PATH"
        error_exit "Failed to move the new AppImage. Restored the original."
    }

    # Make the new AppImage executable
    verbose "Making the new AppImage executable."
    chmod +x "$DEST_APPIMAGE_PATH" || {
        # Restore backup if chmod fails
        mv "$DEST_APPIMAGE_PATH" "$BACKUP_PATH"
        mv "$BACKUP_PATH" "$SELECTED_APPIMAGE_PATH"
        error_exit "Failed to make the new AppImage executable. Restored the original."
    }

    # Determine the new application name
    # Attempt to extract the name from the new AppImage metadata
    if "$DEST_APPIMAGE_PATH" --appimage-info >/dev/null 2>&1; then
        APP_INFO=$("$DEST_APPIMAGE_PATH" --appimage-info 2>/dev/null || true)
        if [ -n "$APP_INFO" ]; then
            APP_NAME_NEW=$(echo "$APP_INFO" | grep '^name=' | head -n1 | cut -d'=' -f2-)
        else
            APP_NAME_NEW=""
        fi
    else
        APP_NAME_NEW=""
    fi

    if [ -z "$APP_NAME_NEW" ]; then
        # Fallback: use the filename without extension
        APP_NAME_NEW="${APPIMAGE_NAME_NEW%.AppImage}"
        verbose "Could not extract application name. Using filename without extension: $APP_NAME_NEW"
    else
        verbose "Extracted application name: $APP_NAME_NEW"
    fi

    # Prompt for application name if still not determined or user wants to change
    read -p "Enter the application name (leave blank to use '$APP_NAME_NEW'): " USER_APP_NAME
    if [ -n "$USER_APP_NAME" ]; then
        APP_NAME_NEW="$USER_APP_NAME"
    elif [ -z "$APP_NAME_NEW" ]; then
        APP_NAME_NEW="${APPIMAGE_NAME_NEW%.AppImage}"
        verbose "Using filename without extension as application name: $APP_NAME_NEW"
    fi

    # Determine the exec command
    EXEC_CMD_NEW="$DEST_APPIMAGE_PATH"

    # Determine the icon path from existing .desktop file
    if [ -f "$SELECTED_DESKTOP_FILE_PATH" ]; then
        ICON_PATH=$(grep "^Icon=" "$SELECTED_DESKTOP_FILE_PATH" | cut -d'=' -f2-)
    else
        ICON_PATH=""
        verbose "No existing .desktop file found. No icon will be set."
    fi

    # Create/update the .desktop file
    create_desktop_file "$APP_NAME_NEW" "$EXEC_CMD_NEW" "$ICON_PATH" "AppImage application: $APP_NAME_NEW"

    # Update the database
    update_database_entry "$SELECTED_APP_NAME" "$EXEC_CMD_NEW" "$SELECTED_DESKTOP_FILE_PATH" "$ICON_PATH"

    # Optionally, remove backup after successful update
    read -p "Do you want to remove the backup of the old AppImage? (y/N): " REMOVE_BACKUP
    case "$REMOVE_BACKUP" in
        y|Y )
            verbose "Removing backup file: $BACKUP_PATH"
            rm -f "$BACKUP_PATH" || verbose "Failed to remove backup file: $BACKUP_PATH"
            ;;
        * )
            verbose "Backup file retained at: $BACKUP_PATH"
            ;;
    esac

    # Update desktop database
    update_desktop_db

    verbose "AppImage updated successfully."
}

# Function to delete an AppImage
delete_appimage() {
    echo "=== Delete an AppImage ==="

    # Select an AppImage
    select_appimage || error_exit "No AppImages available to delete."
    IFS='|' read -r SELECTED_APPIMAGE_PATH SELECTED_DESKTOP_FILE_PATH SELECTED_ICON_PATH <<< "$SELECTED_APP_INFO"

    verbose "Selected AppImage: $SELECTED_APP_NAME"

    # Confirm deletion
    read -p "Are you sure you want to delete '$SELECTED_APP_NAME' and its associated .desktop file? (y/N): " CONFIRM
    case "$CONFIRM" in
        y|Y )
            ;;
        * )
            echo "Delete operation canceled."
            return 0
            ;;
    esac

    # Delete the AppImage
    verbose "Deleting AppImage: $SELECTED_APPIMAGE_PATH"
    rm -f "$SELECTED_APPIMAGE_PATH" || error_exit "Failed to delete AppImage: $SELECTED_APPIMAGE_PATH"

    # Delete the .desktop file
    if [ -f "$SELECTED_DESKTOP_FILE_PATH" ]; then
        verbose "Deleting .desktop file: $SELECTED_DESKTOP_FILE_PATH"
        rm -f "$SELECTED_DESKTOP_FILE_PATH" || error_exit "Failed to delete .desktop file: $SELECTED_DESKTOP_FILE_PATH"
    else
        verbose "No .desktop file found to delete."
    fi

    # Remove from database
    remove_from_database "$SELECTED_APP_NAME"

    # Update desktop database
    update_desktop_db

    verbose "AppImage and its .desktop file have been deleted successfully."
}

# Function to rescan the system and update the database
rescan_system() {
    echo "=== Rescan and Update Database ==="
    read -p "This will scan common directories for AppImages and .desktop files and update the database accordingly. Continue? (y/N): " CONFIRM
    case "$CONFIRM" in
        y|Y )
            ;;
        * )
            echo "Rescan operation canceled."
            return 0
            ;;
    esac

    initialize_database
    load_database
    verbose "Database updated successfully."
}

# Function to display the main menu
show_menu() {
    echo "=== AppImage Manager ==="
    echo "Please select an option:"
    echo "  [1] Add a New AppImage"
    echo "  [2] Edit an Existing AppImage"
    echo "  [3] Update an AppImage Version"
    echo "  [4] Delete an AppImage"
    echo "  [5] Rescan and Update Database"
    echo "  [6] Exit"
}

# Function to handle initial setup
initial_setup() {
    if [ ! -f "$DATABASE_FILE" ]; then
        initialize_database
    fi
}

# Main Script Execution
main() {
    initial_setup
    load_database

    while true; do
        show_menu
        read -p "Enter your choice [1-6]: " MENU_CHOICE
        case "$MENU_CHOICE" in
            1)
                add_appimage
		        load_database
                ;;
            2)
                edit_appimage
		        load_database
                ;;
            3)
                update_appimage
		        load_database
                ;;
            4)
                delete_appimage
		        load_database
                ;;
            5)
                rescan_system
		        load_database
                ;;
            6)
                echo "Exiting AppImage Manager. Goodbye!"
                exit 0
                ;;
            *)
                echo "Invalid choice. Please enter a number between 1 and 6."
                ;;
        esac
	    load_database
        echo ""
    done
}

# Execute the main function
main