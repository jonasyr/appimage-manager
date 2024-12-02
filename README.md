# AppImage Manager

A comprehensive Bash script to manage AppImage files on your Linux system.

## Overview

AppImage Manager simplifies the management of AppImage files by providing an interactive interface to:

- **Add New AppImages**: Easily add and integrate AppImages into your system.
- **Edit Existing AppImages**: Modify application names, icons, and executable paths.
- **Update AppImage Versions**: Replace old versions with new ones seamlessly.
- **Delete AppImages**: Remove AppImages and their associated `.desktop` files.
- **Automatic Handling**: Manages file locations, permissions, and desktop entries for you.

## Installation

1. **Clone the Repository**:

   ~~~bash
   git clone https://github.com/jonasyr/appimage-manager.git
   ~~~

2. **Navigate to the Directory**:

   ~~~bash
   cd appimage-manager
   ~~~

3. **Make the Script Executable**:

   ~~~bash
   chmod +x handle_appimage.sh
   ~~~

4. **Move the Script to a Directory in Your PATH** (e.g., `/usr/local/bin`):

   ~~~bash
   sudo mv handle_appimage.sh /usr/local/bin/appimage-manager
   ~~~

   > **Note**: You may need to enter your password for `sudo`.

5. **Verify Installation**:

   ~~~bash
   appimage-manager
   ~~~

## Usage

Run the script by typing:

~~~bash
appimage-manager
~~~

You will be presented with a menu of options:
~~~bash
=== AppImage Manager ===
Please select an option:
  [1] Add a New AppImage
  [2] Edit an Existing AppImage
  [3] Update an AppImage Version
  [4] Delete an AppImage
  [5] Rescan and Update Database
  [6] Exit
Enter your choice [1-6]:
~~~

### Adding a New AppImage

1. **Select Option 1**: Add a New AppImage.
2. **Follow the Prompts**:
   - Enter the path to the AppImage file (it will attempt to detect recent downloads in `~/Downloads`).
   - Specify an icon file (optional).
   - Confirm the application name (it tries to extract it from the AppImage metadata).
3. **Completion**: The script moves the AppImage to `~/Applications`, sets executable permissions, and creates a `.desktop` file in `~/.local/share/applications`.

### Editing an Existing AppImage

1. **Select Option 2**: Edit an Existing AppImage.
2. **Choose the AppImage** from the list provided.
3. **Select What to Edit**:
   - Application Name
   - Icon
   - Executable Path
   - All of the Above
4. **Follow the Prompts** to make the desired changes.

### Updating an AppImage Version

1. **Select Option 3**: Update an AppImage Version.
2. **Choose the AppImage** you wish to update.
3. **Provide the Path** to the new AppImage version.
4. **Confirmation**: The script backs up the old AppImage and replaces it with the new one.

### Deleting an AppImage

1. **Select Option 4**: Delete an AppImage.
2. **Choose the AppImage** you wish to delete.
3. **Confirmation**: The script will remove the AppImage file and its associated `.desktop` file.

### Rescanning and Updating the Database

1. **Select Option 5**: Rescan and Update Database.
2. **Confirmation**: The script scans common directories for AppImages and updates its internal database.

## Configuration

The script uses the following directories by default:

- **AppImages Directory**: `~/Applications`
- **Icons Directory**: `~/Icons`
- **Desktop Files Directory**: `~/.local/share/applications`
- **Configuration Directory**: `~/.config/handle_appimage`
- **Database File**: `~/.config/handle_appimage/appimages.csv`

You can change these directories by modifying the constants at the top of the script.

## Contributing

Contributions are welcome! Please fork the repository and submit a pull request with your improvements.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Support

If you encounter any issues or have questions, feel free to open an [issue](https://github.com/yourusername/appimage-manager/issues) on GitHub.

---




