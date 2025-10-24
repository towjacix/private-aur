Alright, let's get you the **full, soup-to-nuts guide** for a completely automated private binary AUR repo with GitHub Actions! This will handle everything from checking for `PKGBUILD` updates to building and deploying your binaries.

Here's the plan:

## 1️⃣ GitHub Repository Setup

First things first, let's get your GitHub repository ready.

1.  **Create a New Private GitHub Repository:**
    *   Go to [github.com/new](https://github.com/new).
    *   Give it a name like `aur-binary-repo` (or whatever you prefer).
    *   Make sure it's **Private**.
    *   Initialize it with a README (optional, but good practice).

2.  **Clone Your New Repository Locally:**
    ```bash
    git clone https://github.com/<your-username>/aur-binary-repo.git
    cd aur-binary-repo
    ```

3.  **Set Up Initial Directory Structure:**
    You'll need a `packages/` directory where each subdirectory will hold the `PKGBUILD` for an AUR package you want to track.
    ```bash
    mkdir -p packages/.github/workflows
    ```

4.  **Add Your First PKGBUILD (Example: `neovide`):**
    *   Go to the AUR page for `neovide` (or your chosen package).
    *   Find the "View PKGBUILD" link and copy its raw content.
    *   Create the package directory and paste the `PKGBUILD` content:
        ```bash
        mkdir packages/neovide
        # Create the file and paste the PKGBUILD content into it
        nano packages/neovide/PKGBUILD
        ```
    *   Repeat this for any other packages you want to include initially.

5.  **Initial Commit:**
    ```bash
    git add .
    git commit -m "Initial repository setup and added neovide PKGBUILD"
    git push origin main
    ```

## 2️⃣ GPG Key Management (for Signing)

You need a GPG key to sign your packages and the repository database. This ensures integrity and authenticity.

1.  **Generate a GPG Key (if you don't have one):**
    *   Open your terminal and run:
        ```bash
        gpg --full-generate-key
        ```
    *   Follow the prompts:
        *   Choose `RSA and RSA`.
        *   Keysize: `4096` bits (more secure).
        *   Key validity: Set an expiration date or none (e.g., `0` for no expiration).
        *   Real name: Your name.
        *   Email address: Your email.
        *   Comment: (Optional)
        *   **Crucially, set a strong passphrase!** You'll need this later.
    *   **Note down your GPG Key ID.** You can find it with `gpg --list-secret-keys --keyid-format LONG`. It looks like `0xABCDEF1234567890`.

2.  **Export Your Private GPG Key:**
    *   This is the sensitive part. **Do not share this file or commit it to Git!**
    *   Run:
        ```bash
        gpg --export-secret-keys --armor <YOUR_GPG_KEY_ID> > private.key
        ```
        Replace `<YOUR_GPG_KEY_ID>` with your actual GPG key ID (e.g., `0xABCDEF1234567890`).
    *   You'll be prompted for your GPG passphrase.

3.  **Create a GitHub Secret for Your Private Key:**
    *   Go to your GitHub repository on the web.
    *   Navigate to **Settings > Secrets and variables > Actions**.
    *   Click **New repository secret**.
    *   **Name:** `GPG_PRIVATE_KEY`
    *   **Value:** Open the `private.key` file you just created with a text editor (`cat private.key`). Copy its **entire content**, including the `-----BEGIN PGP PRIVATE KEY BLOCK-----` and `-----END PGP PRIVATE KEY BLOCK-----` lines, and paste it into the value field.
    *   Click **Add secret**.
    *   **Delete the `private.key` file from your local machine now.** It's safely stored in GitHub Secrets.

4.  **Export Your Public GPG Key:**
    *   This key is public and will be distributed with your repo so `pacman` can verify signatures.
    *   Run:
        ```bash
        gpg --export --armor <YOUR_GPG_KEY_ID> > public-key.asc
        ```
        Replace `<YOUR_GPG_KEY_ID>`.
    *   You can keep this file for now; the GitHub Action will eventually upload it to the release.

## 3️⃣ GitHub Actions Workflow: Automated PKGBUILD Updater

This workflow will periodically check the AUR for updates to your `PKGBUILD`s and automatically push those changes to your repository.

1.  **Create the Workflow File:**
    Create a file named `.github/workflows/update-pkgbuilds.yml` in your repository:
    ```yaml
    # .github/workflows/update-pkgbuilds.yml
    name: Automated PKGBUILD Updater

    on:
      schedule:
        - cron: '0 0 * * *' # Run daily at midnight UTC. Adjust as needed (e.g., '0 */6 * * *' for every 6 hours)
      workflow_dispatch: # Allows manual triggering from GitHub Actions tab

    jobs:
      update-pkgbuilds:
        runs-on: ubuntu-latest

        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Install Git
            run: |
              sudo apt-get update
              sudo apt-get install -y git

          - name: Update PKGBUILDs from AUR
            id: update_pkgs
            run: |
              # Set a flag to track if any changes were made
              CHANGES_MADE=false

              # Loop through each package directory in your 'packages/' folder
              for pkgdir in packages/*; do
                pkgname=$(basename "$pkgdir")
                echo "Checking $pkgname for updates..."

                # Clone the official AUR repository for the package
                # Use a temporary directory to avoid conflicts
                git clone https://aur.archlinux.org/"$pkgname".git temp_aur_clone

                if [ -d "temp_aur_clone" ]; then
                  # Compare the PKGBUILD from AUR with our local one
                  if ! diff -q "$pkgdir"/PKGBUILD temp_aur_clone/PKGBUILD > /dev/null; then
                    echo "  -> PKGBUILD for $pkgname has changed in AUR. Updating..."
                    cp temp_aur_clone/PKGBUILD "$pkgdir"/PKGBUILD
                    git add "$pkgdir"/PKGBUILD
                    CHANGES_MADE=true
                  else
                    echo "  -> PKGBUILD for $pkgname is up-to-date."
                  fi
                  rm -rf temp_aur_clone
                else
                  echo "  -> Warning: Could not clone AUR repo for $pkgname. Skipping."
                fi
              done

              # Set output to indicate if changes were made
              echo "changes_made=$CHANGES_MADE" >> "$GITHUB_OUTPUT"

          - name: Commit and Push if changes were made
            if: steps.update_pkgs.outputs.changes_made == 'true'
            run: |
              git config user.name "GitHub Actions Bot"
              git config user.email "actions@github.com"
              git commit -m "Automated: Update PKGBUILDs from AUR"
              git push
            env:
              GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Automatically provided by GitHub Actions
    ```
2.  **Commit and Push this workflow:**
    ```bash
    git add .github/workflows/update-pkgbuilds.yml
    git commit -m "Add automated PKGBUILD updater workflow"
    git push origin main
    ```

## 4️⃣ GitHub Actions Workflow: Automated Binary Build & Deploy

This workflow will build your packages, sign them, update the repository database, and publish everything to a GitHub Release.

1.  **Create the Workflow File:**
    Create a file named `.github/workflows/build-repo.yml` in your repository:
    ```yaml
    # .github/workflows/build-repo.yml
    name: Build and Deploy AUR Binary Repo

    on:
      push:
        branches:
          - main
        paths:
          - 'packages/**' # Trigger only if PKGBUILDs or related package files change
      workflow_dispatch: # Allows manual triggering

    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest # Use a standard Ubuntu runner

        container:
          image: archlinux/archlinux:latest # Use an Arch Linux Docker image
          options: --privileged # Needed for systemd-nspawn/chroot if using aurutils

        steps:
          - name: Checkout repository
            uses: actions/checkout@v4

          - name: Install build dependencies
            run: |
              pacman -Sy --noconfirm
              pacman -S --noconfirm base-devel devtools git gnupg aurutils # aurutils simplifies chroot builds

          - name: Import GPG private key
            run: |
              echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --batch --import
              # Trust the key (optional, but prevents "no ultimately trusted keys" warnings)
              # Replace <YOUR_GPG_KEY_ID_HERE> with your actual GPG Key ID (e.g., 0xABCDEF1234567890)
              echo -e "trust\n5\ny\n" | gpg --batch --command-fd 0 --edit-key <YOUR_GPG_KEY_ID_HERE>
            env:
              YOUR_GPG_KEY_ID_HERE: <YOUR_GPG_KEY_ID_HERE> # Replace with your GPG key ID

          - name: Configure makepkg for signing
            run: |
              # Replace <YOUR_GPG_KEY_ID_HERE> with your actual GPG Key ID
              sed -i 's/^#GPGKEY=""/GPGKEY="<YOUR_GPG_KEY_ID_HERE>"/' /etc/makepkg.conf
              # Ensure BUILDENV contains 'sign' for automatic signing by makepkg
              sed -i 's/^BUILDENV=(.*)/BUILDENV=(fakeroot !distcc color ccache check sign)/' /etc/makepkg.conf
            env:
              YOUR_GPG_KEY_ID_HERE: <YOUR_GPG_KEY_ID_HERE> # Replace with your GPG key ID

          - name: Prepare repository directory
            run: |
              mkdir -p repo/x86_64
              # Export public key to include in the release
              # Replace <YOUR_GPG_KEY_ID_HERE> with your actual GPG Key ID
              gpg --export --armor <YOUR_GPG_KEY_ID_HERE> > repo/public-key.asc
            env:
              YOUR_GPG_KEY_ID_HERE: <YOUR_GPG_KEY_ID_HERE>

          - name: Build and add packages to repository
            run: |
              # Initialize aurutils repository (creates repo.db.tar.gz if it doesn't exist)
              # -d repo: specifies the directory where the repository files will be stored
              # -c: creates a clean chroot for each build
              # -r repo: specifies the name of the repository (used internally by aurutils)
              # -s: signs the built packages
              # -u: updates the repository database after adding packages
              aur sync -d repo -c -r repo -s -u

              for pkgdir in packages/*; do
                pkgname=$(basename "$pkgdir")
                echo "Building $pkgname..."
                cd "$pkgdir"
                # 'aur build' builds the package in a clean chroot and adds it to the specified repo
                # -c: ensures a clean chroot
                # -d ../../repo: specifies the repository directory relative to the current working directory
                # -r repo: specifies the repo name
                # -s: signs the package
                # -u: updates the repository database
                aur build -c -d ../../repo -r repo -s -u
                cd - # Go back to the root of the repository
              done
            env:
              AURDEST: ${{ github.workspace }}/repo # aurutils uses AURDEST for the repo directory

          - name: Create or update GitHub Release
            uses: softprops/action-gh-release@v1
            with:
              tag_name: latest # Always update the 'latest' release
              name: Latest AUR Binary Packages
              body: |
                Automated build of AUR packages.
                Built on ${{ github.event.head_commit.timestamp }}
              draft: false
              prerelease: false
              files: |
                repo/x86_64/*.pkg.tar.zst
                repo/x86_64/*.pkg.tar.zst.sig
                repo/repo.db.tar.gz
                repo/repo.db.tar.gz.sig
                repo/repo.files.tar.gz
                repo/repo.files.tar.gz.sig
                repo/public-key.asc
            env:
              GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # Automatically provided by GitHub Actions
    ```
2.  **Replace Placeholders:**
    *   **Crucially, replace all instances of `<YOUR_GPG_KEY_ID_HERE>`** with your actual GPG Key ID (e.g., `0xABCDEF1234567890`).

3.  **Commit and Push this workflow:**
    ```bash
    git add .github/workflows/build-repo.yml
    git commit -m "Add automated binary build and deploy workflow"
    git push origin main
    ```

## 5️⃣ Client-Side Setup (Your Arch Linux Machine)

Now, on your Arch Linux system, you need to configure `pacman` to use this shiny new automated repository.

1.  **Import Your Public GPG Key:**
    *   Go to your GitHub repository on the web.
    *   Click on the **"Releases"** tab.
    *   You should see a release named `latest` (it might take a few minutes for the GitHub Action to run the first time).
    *   Under "Assets", download the `public-key.asc` file.
    *   On your Arch machine, run:
        ```bash
        sudo pacman-key --add /path/to/downloaded/public-key.asc
        sudo pacman-key --lsign-key <YOUR_GPG_KEY_ID>
        ```
        Replace `/path/to/downloaded/public-key.asc` with the actual path and `<YOUR_GPG_KEY_ID>` with your GPG Key ID.

2.  **Configure Pacman:**
    *   Edit your `/etc/pacman.conf` file (e.g., `sudo nano /etc/pacman.conf`).
    *   Add the following section to the **very end** of the file:
        ```
        [myrepo]
        SigLevel = PackageRequired
        Server = https://github.com/<YOUR_GITHUB_USERNAME>/aur-binary-repo/releases/download/latest/$arch
        ```
        *   Replace `<YOUR_GITHUB_USERNAME>` with your actual GitHub username.
        *   `[myrepo]` is the name of your repository.
        *   `SigLevel = PackageRequired` is highly recommended for security.

3.  **Update Pacman's Database:**
    ```bash
    sudo pacman -Sy
    ```
    You should now see `myrepo` being synchronized!

## 🎉 You're Done! The Automated Workflow

Here's how it all comes together:

1.  **Scheduled Updates:** Your `update-pkgbuilds.yml` workflow runs daily (or on your chosen schedule).
2.  **PKGBUILD Sync:** It checks the official AUR for updates to the `PKGBUILD`s in your `packages/` directory.
3.  **Automatic Push:** If any `PKGBUILD` is newer on the AUR, it automatically commits and pushes that updated `PKGBUILD` to your `main` branch.
4.  **Binary Build Trigger:** This push to `main` (specifically in the `packages/` path) automatically triggers your `build-repo.yml` workflow.
5.  **Build & Deploy:** The `build-repo.yml` workflow:
    *   Sets up an Arch Linux environment.
    *   Imports your GPG private key (securely from GitHub Secrets).
    *   Builds the updated package(s) in a clean chroot using `aurutils`.
    *   Signs the built packages and the repository database.
    *   Updates the `latest` GitHub Release with the new binaries and repository files.
6.  **Client Update:** On your Arch Linux machine, when you run `sudo pacman -Syu`, it will fetch the latest packages from your GitHub Release, signed and ready to install!

Now you have a fully automated system for keeping your private AUR binary repository up-to-date without lifting a finger (after the initial setup, of course!). Enjoy!
