# Xin
A Packaged Package Manager TUI developed in Golang using Bubble Tea for Debian-based systems.

I plan to develop this because I like installing packages with `--no-install-recommends --no-install-suggests` in `apt` and `nala` or `-R` in `aptitude`. It's bad in regular practice because recommends are somewhat crucial (not as crucial as depends).

This can be a good start to building out a completely minimal system without all the unnecessary bloat if the user knows that a certain feature of the app are almost never used.

Using `apt` as the main package manager for the example. As a start, the workflow will be something like this:

1. `apt xinstall vim` or `apt xin vim` will install vim and its dependency but doesn't install the recommends and suggests.
2. Xin will then open another prompt after installation has succeed for user to select which from the recommends and suggests list that the user would like to include it with. The options will be Install Everything, Install Selected, or Skip.
3. Any further installation, meaning for each of selected recommends and/or suggests will also go through Xin and the process repeats.

It's a simple process, but eventually Xin will be able to manage all recommends and suggests, as well as pointing out which packages recommends/suggests each of the packaged packages, and allowing installation, removal, and purge. 

# Why Xin?

Well, I ran out of whatever I want to call it so Install and NONO (X) gave birth to Xin. I used to know someone that goes by Xin... ahhh they went quiet.

# How?

1. Well, as I mentioned once it receives a package name it'll install it with a flag depending on which package manager you chose to install that package with.

Nala
`sudo nala install <package> -y --no-install-recommends --no-install-suggests`

Apt
`sudo apt install <package> -y --no-install-recommends --no-install-suggests`

Aptitude
`sudo aptitude install <package> -y --R`

2. Then it'll run another command to get a list of not-yet installed recommends:
`apt-cache depends <PACKAGE> | awk '/^[ |]*Recommends:/{p=1; if ($0 !~ /</) {sub(/^[ |]*Recommends: /,""); if($0!="") print} next} /^  [A-Z]/{p=0} p && /^    /{sub(/^    /,""); if($0!="") print}' | sed 's/:.*//' | sort -u | while read pkg; do dpkg-query -W -f='${Status}' "$pkg" 2>/dev/null | grep -q "install ok installed" || echo "$pkg"; done`

3. And a similar command for suggests:
`apt-cache depends <PACKAGE> | awk '/^[ |]*Suggests:/{p=1; if ($0 !~ /</) {sub(/^[ |]*Suggests: /,""); if($0!="") print} next} /^  [A-Z]/{p=0} p && /^    /{sub(/^    /,""); if($0!="") print}' | sed 's/:.*//' | sort -u | while read pkg; do dpkg-query -W -f='${Status}' "$pkg" 2>/dev/null | grep -q "install ok installed" || echo "$pkg"; done`

4. Then it'll get the description for each packages as user context:
`apt show <PACKAGE> | awk -F': ' '/^Description:/ {print $2; exit}'`

That's about it.

# Prototype

I have a prototype running on my local machine using Whiptail as the TUI. I would polish it but I decided to turn it into a bigger project so I never bothered. If you feel like testing it, just paste this code at the bottom of your `~/.bashrc`:

```bash
export NEWT_COLORS='
    root=green,black
    border=green,black
    title=green,black
    roottext=white,black
    window=green,black
    textbox=white,black
    button=black,green
    compactbutton=white,black
    listbox=white,black
    actlistbox=black,white
    actsellistbox=black,green
    checkbox=green,black
    actcheckbox=black,green
    suggests=green,black
'
# Function to detect available package manager
detect_package_manager() {
    if command -v nala >/dev/null 2>&1; then
        echo "nala"
    elif command -v aptitude >/dev/null 2>&1; then
        echo "aptitude"
    elif command -v apt >/dev/null 2>&1; then
        echo "apt"
    else
        echo "apt"  # fallback
    fi
}
# Function to get uninstalled recommended packages using apt-cache
get_uninstalled_recommends() {
    local package="$1"
    apt-cache depends "$package" | awk '/^[ |]*Recommends:/{p=1; if ($0 !~ /</) {sub(/^[ |]*Recommends: /,""); if($0!="") print} next} /^  [A-Z]/{p=0} p && /^    /{sub(/^    /,""); if($0!="") print}' | sed 's/:.*//' | sort -u | while read pkg; do 
        if [ -n "$pkg" ]; then
            dpkg-query -W -f='${Status}' "$pkg" 2>/dev/null | grep -q "install ok installed" || echo "$pkg"
        fi
    done
}
# Function to get uninstalled suggested packages using apt-cache
get_uninstalled_suggests() {
    local package="$1"
    apt-cache depends "$package" | awk '/^[ |]*Suggests:/{p=1; if ($0 !~ /</) {sub(/^[ |]*Suggests: /,""); if($0!="") print} next} /^  [A-Z]/{p=0} p && /^    /{sub(/^    /,""); if($0!="") print}' | sed 's/:.*//' | sort -u | while read pkg; do 
        if [ -n "$pkg" ]; then
            dpkg-query -W -f='${Status}' "$pkg" 2>/dev/null | grep -q "install ok installed" || echo "$pkg"
        fi
    done
}
# Function to get package description
get_package_description() {
    local package="$1"
    aptitude show "$package" 2>/dev/null | awk -F': ' '/^Description:/ {print $2; exit}' || echo "No description available"
}
# Function to check if package exists and is not virtual
is_real_package() {
    local package="$1"
    apt-cache show "$package" 2>/dev/null | grep -q '^Version:'
}
# Main install-selective function
xin() {
    local package="$1"
    local package_manager="${2:-$(detect_package_manager)}"
    # Check if package name is provided
    if [ -z "$package" ]; then
        echo "Usage: install-selective <package> [package_manager]"
        return 1
    fi
    echo "Using package manager: $package_manager"
    # Check if package is already installed
    local is_installed=false
    if dpkg-query -W -f='${Status}' "$package" 2>/dev/null | grep -q "ok installed"; then
        is_installed=true
        echo "Package '$package' is already installed."
    else
        echo "Installing package: $package using $package_manager"
        # Install the main package with appropriate flags based on package manager
        case "$package_manager" in
            apt)
                if ! sudo apt install "$package" --no-install-recommends --no-install-suggests -y; then
                    echo "Failed to install $package"
                    return 1
                fi
                ;;
            nala)
                if ! sudo nala install "$package" --no-install-recommends --no-install-suggests -y; then
                    echo "Failed to install $package"
                    return 1
                fi
                ;;
            aptitude)
                if ! sudo aptitude install "$package" -y -R; then
                    echo "Failed to install $package"
                    return 1
                fi
                ;;
            *)
                echo "Unsupported package manager: $package_manager"
                return 1
                ;;
        esac
        echo "Package '$package' installed successfully."
    fi
    # Get list of not-yet-installed recommended and suggested packages
    echo "Checking for recommended packages..."
    local recommends_list
    recommends_list=$(get_uninstalled_recommends "$package")
    echo "Checking for suggested packages..."
    local suggests_list
    suggests_list=$(get_uninstalled_suggests "$package")
    # Combine recommends and suggests
    local all_packages_list
    all_packages_list=$(printf "%s\n%s" "$recommends_list" "$suggests_list" | sort -u | grep -v '^$')
    # If no packages to install, exit
    if [ -z "$all_packages_list" ]; then
        echo "No uninstalled recommended or suggested packages found for '$package'."
        return 0
    fi
    # Create whiptail checklist options
    local checklist_options=()
    local valid_packages=()
    while IFS= read -r rec_pkg; do
        if [ -n "$rec_pkg" ]; then
            # Check if the package is real and not virtual
            if ! is_real_package "$rec_pkg"; then
                echo "Info: Skipping virtual or non-existent package: '$rec_pkg'" >&2
                continue
            fi
            # Determine if it's recommended or suggested
            local package_type=""
            if echo "$recommends_list" | grep -q "^$rec_pkg$"; then
                package_type="[R] "
            elif echo "$suggests_list" | grep -q "^$rec_pkg$"; then
                package_type="[S] "
            fi
            # Get package description
            local desc
            desc=$(get_package_description "$rec_pkg")
            # If description is empty, use a default
            if [ -z "$desc" ]; then
                desc="No description available"
            fi
            # Truncate description if too long (for display purposes)
            if [ ${#desc} -gt 50 ]; then
                desc="${desc:0:47}..."
            fi
            # Add package type prefix to description
            desc="${package_type}${desc}"
            # Add to checklist options (package, description, off)
            checklist_options+=("$rec_pkg" "$desc" "off")
            valid_packages+=("$rec_pkg")
        fi
    done <<< "$all_packages_list"
    # If no valid packages found, exit
    if [ ${#checklist_options[@]} -eq 0 ]; then
        echo "No valid recommended or suggested packages found."
        return 0
    fi
    # Show whiptail checklist
    local selected_packages
    selected_packages=$(whiptail --title "Recommended [R] and Suggested [S] Packages for '$package'" \
        --checklist "Select packages to install (Space to select, Tab to navigate):" \
        20 80 10 \
        "${checklist_options[@]}" \
        3>&1 1>&2 2>&3)
    # Check if user cancelled
    if [ $? -ne 0 ]; then
        echo "Installation cancelled by user."
        return 0
    fi
    # If no packages selected, ask for bulk actions
    if [ -z "$selected_packages" ]; then
        local choice
        choice=$(whiptail --title "No Packages Selected" \
            --menu "What would you like to do?" \
            15 60 4 \
            "install_all" "Install All Recommended & Suggested" \
            "install_rec" "Install Only Recommended" \
            "skip" "Skip Installation" \
            3>&1 1>&2 2>&3)
        case $choice in
            "install_all")
                selected_packages=""
                for pkg in "${valid_packages[@]}"; do
                    selected_packages="$selected_packages \"$pkg\""
                done
                ;;
            "install_rec")
                selected_packages=""
                while IFS= read -r rec_pkg; do
                    if [ -n "$rec_pkg" ] && is_real_package "$rec_pkg"; then
                        selected_packages="$selected_packages \"$rec_pkg\""
                    fi
                done <<< "$recommends_list"
                ;;
            "skip"|*)
                echo "Skipping packages installation."
                return 0
                ;;
        esac
    fi
    # Parse selected packages (remove quotes)
    local packages_to_install
    packages_to_install=$(echo "$selected_packages" | tr -d '"')
    if [ -n "$packages_to_install" ]; then
        echo "Installing selected packages..."
        # Install each selected package using install-selective (recursive)
        for pkg in $packages_to_install; do
            # Check if package is virtual before attempting to install
            if ! is_real_package "$pkg"; then
                echo "Info: Skipping install of virtual package: '$pkg'" >&2
                continue
            fi
            echo ""
            echo "Installing package: $pkg"
            # Recursively call install-selective for each package
            install-selective "$pkg" "$package_manager"
        done
        echo ""
        echo "Finished installing extra packages for '$package'."
    else
        echo "No packages selected for installation."
    fi
}

apt_wrapper() {
  backend="$1"; shift
  APT_PATH=$(which apt)
  NALA_PATH=$(which nala)
  APTITUDE_PATH=$(which aptitude)
  case "$1" in
    xin)
      shift
      for package in "$@"; do
        if [[ ! "$package" =~ ^- ]]; then
          xin "$package" "$backend"
        fi
      done
      ;;
    *)
      case "$backend" in
        nala)
          command sudo "$NALA_PATH" "$@"
          ;;
        aptitude)
          command sudo "$APTITUDE_PATH" "$@"
          ;;
        apt|*)
          command sudo "$APT_PATH" "$@"
          ;;
      esac
      ;;
  esac
}

apt() {
  apt_wrapper apt "$@"
}

nala() {
  apt_wrapper nala "$@"
}

aptitude() {
  apt_wrapper aptitude "$@"
}
```
