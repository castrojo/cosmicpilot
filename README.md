# cosmicpilot

A custom bootable operating system based on [Universal Blue](https://universal-blue.org/) and [Bluefin](https://projectbluefin.io), featuring System76's COSMIC Desktop environment instead of GNOME. This is a production-ready bootc image optimized for modern development workflows.

> Explore the cosmos with your own custom Linux desktop.

## What's Included

### COSMIC Desktop Environment
- System76's modern COSMIC desktop built in Rust
- Native Wayland support with cosmic-comp compositor
- COSMIC Panel, Launcher, Settings, and Applications
- Cosmic Terminal, Files, and Text Editor
- Replaces GNOME with a fresh, performant desktop experience

### Build System
- Automated builds via GitHub Actions on every commit
- Self-hosted Renovate setup that keeps all your images and actions up to date
- Automatic cleanup of old images (90+ days) to keep it tidy
- Pull request workflow - test changes before merging to main
  - PRs build and validate before merge
  - `main` branch builds `:stable` images
- Validates your files on pull requests so you never break a build:
  - Brewfile, Justfile, ShellCheck, Renovate config, and Flatpak validation
- Production Grade Features
  - Container signing, SBOM Generation, and layer rechunking
  - See checklist below to enable these as they take some manual configuration

### Homebrew Integration
- Pre-configured Brewfiles for easy package installation and customization
- Includes curated collections: development tools, fonts, CLI utilities
- Users install packages at runtime with `brew bundle`, aliased to premade `ujust` commands
- See [custom/brew/](custom/brew/) for details

### Flatpak Support
- Ship your favorite flatpaks
- Automatically installed on first boot after user setup
- See [custom/flatpaks/](custom/flatpaks/) for details

### Rechunker
- Optimizes container image layer distribution for better download resumability
- Based on [hhd-dev/rechunk](https://github.com/hhd-dev/rechunk) v1.2.4
- Disabled by default for faster initial builds
- Enable in `.github/workflows/build.yml` by uncommenting the rechunker steps
- Recommended for production deployments after initial testing

### ujust Commands
- User-friendly command shortcuts via `ujust`
- Pre-configured examples for app installation and system maintenance
- See [custom/ujust/](custom/ujust/) for details

### Build Scripts
- Modular numbered scripts (10-, 20-, 30-) run in order
- COSMIC desktop installation via `30-cosmic-desktop.sh`
- Helper functions for safe COPR usage
- See [build/](build/) for details

## Quick Start

### 1. Enable GitHub Actions

**IMPORTANT: Do this first before making any other changes!**

- Go to the "Actions" tab in your repository on GitHub
- Click "I understand my workflows, go ahead and enable them"

Your first build will start automatically!

Note: Image signing is disabled by default. Your images will build successfully without any signing keys. Once you're ready for production, see "Optional: Enable Image Signing" below.

### 2. Wait for Initial Build

The first build takes about 15-20 minutes. You can monitor progress in the Actions tab.

### 3. Deploy Your Image

Once the build completes, switch to your custom COSMIC image:

```bash
sudo bootc switch ghcr.io/castrojo/cosmicpilot:stable
sudo systemctl reboot
```

After reboot, select the "COSMIC" session at the login screen.

### 4. Customize Your Image

Customize your apps:
- Add Brewfiles in `custom/brew/`
- Add Flatpaks in `custom/flatpaks/`
- Add ujust commands in `custom/ujust/`

Add packages in `build/10-build.sh`:
```bash
dnf5 install -y package-name
```

### 5. Development Workflow

All changes should be made via pull requests:

1. Open a pull request on GitHub with the change you want
2. The PR will automatically trigger:
   - Build validation
   - Brewfile, Flatpak, Justfile, and shellcheck validation
   - Test image build
3. Once checks pass, merge the PR
4. Merging publishes a `:stable` image

## Optional: Enable Image Signing

Image signing is disabled by default to let you start building immediately. However, signing is strongly recommended for production use.

### Why Sign Images?

- Verify image authenticity and integrity
- Prevent tampering and supply chain attacks
- Required for some enterprise/security-focused deployments
- Industry best practice for production images

### Setup Instructions

#### Step 1: Generate Signing Keys

On your local machine, install cosign and generate keys:

```bash
# Install cosign (if not already installed)
# On Fedora/RHEL:
sudo dnf install cosign

# On other systems, see: https://docs.sigstore.dev/cosign/installation/

# Generate key pair
cosign generate-key-pair
```

This creates two files:
- `cosign.key` (private key) - **Keep this secret!**
- `cosign.pub` (public key) - This will be committed to your repository

#### Step 2: Add Private Key to GitHub Secrets

1. Copy the entire contents of `cosign.key`:
   ```bash
   cat cosign.key
   ```

2. Go to your repository on GitHub
3. Navigate to **Settings** → **Secrets and variables** → **Actions**
4. Click "**New repository secret**"
5. Name: `SIGNING_SECRET`
6. Value: Paste the entire contents of `cosign.key` (including the BEGIN and END lines)
7. Click "**Add secret**"

**Important:** Never commit `cosign.key` to the repository. It's already in `.gitignore`.

#### Step 3: Update Public Key in Repository

1. Replace the contents of `cosign.pub` in your repository:
   ```bash
   cat cosign.pub > cosign.pub
   ```

2. Commit and push the change:
   ```bash
   git add cosign.pub
   git commit -m "feat: add cosign public key"
   git push
   ```

#### Step 4: Enable Signing in Workflow

1. Edit `.github/workflows/build.yml`
2. Find the comment `# OPTIONAL: Image Signing with Cosign` (around line 170)
3. Uncomment the signing steps (remove the `#` from the beginning of each line):
   - "Install Cosign"
   - "Sign container image"
4. Commit and push:
   ```bash
   git add .github/workflows/build.yml
   git commit -m "feat: enable image signing"
   git push
   ```

#### Step 5: Verify Signing Works

Your next build will produce signed images! Users can verify your images with:

```bash
cosign verify --key cosign.pub ghcr.io/castrojo/cosmicpilot:stable
```

## Love Your Image? Let's Go to Production

Ready to take your custom OS to production? Enable these features for enhanced security, reliability, and performance:

### Production Checklist

- [ ] **Enable Image Signing** (Recommended)
  - Provides cryptographic verification of your images
  - Prevents tampering and ensures authenticity
  - See "Optional: Enable Image Signing" section above for setup instructions
  - Status: **Disabled by default** to allow immediate testing

- [ ] **Enable Rechunker** (Recommended)
  - Optimizes image layer distribution for better download resumability
  - Improves reliability for users with unstable connections
  - To enable:
    1. Edit `.github/workflows/build.yml`
    2. Find the "Rechunk (OPTIONAL)" section around line 121
    3. Uncomment the "Run Rechunker" step
    4. Uncomment the "Load in podman and tag" step
    5. Comment out the "Tag for registry" step that follows
    6. Commit and push
  - Status: **Disabled by default** for faster initial builds

- [ ] **Enable SBOM Attestation** (Recommended)
  - Generates Software Bill of Materials for supply chain security
  - Provides transparency about what's in your image
  - Requires image signing to be enabled first
  - To enable:
    1. First complete image signing setup above
    2. Edit `.github/workflows/build.yml`
    3. Find the "OPTIONAL: SBOM Attestation" section
    4. Uncomment the "Setup Syft" and "Generate SBOM" steps
    5. Commit and push
  - Status: **Disabled by default** (requires signing first)

### After Enabling Production Features

Your workflow will:
- Sign all images with your key
- Generate and attach SBOMs
- Optimize layers for better distribution
- Provide full supply chain transparency

## GitHub Setup Requirements

To use this repository effectively, you need to:

1. **Enable GitHub Actions** (Required)
   - Go to Settings → Actions → General
   - Under "Actions permissions", select "Allow all actions and reusable workflows"
   - Under "Workflow permissions", select "Read and write permissions"
   - Click "Save"

2. **Enable GitHub Packages** (Automatically enabled with Actions)
   - Your images will be published to GitHub Container Registry (ghcr.io)
   - Images are public by default

3. **Add Signing Secret** (Optional, for production)
   - See "Optional: Enable Image Signing" section above

4. **Configure Renovate** (Optional, for automated updates)
   - Renovate is pre-configured and will run automatically
   - It creates PRs to update dependencies
   - Review and merge these PRs to keep your system current

## Local Testing

Test your changes before pushing:

```bash
just build              # Build container image
just build-qcow2        # Build VM disk image
just run-vm-qcow2       # Test in browser-based VM
```

## About COSMIC Desktop

[COSMIC](https://github.com/pop-os/cosmic-epoch) is a new desktop environment built in Rust by System76. It features:

- Modern, performant architecture written in Rust
- Native Wayland support
- Tiling window management
- Highly customizable settings
- Resource-efficient design

This image replaces the default GNOME desktop with COSMIC during the build process, giving you a fresh desktop experience while maintaining all the Universal Blue benefits.

## Community

- [Universal Blue Forums](https://universal-blue.discourse.group/)
- [Universal Blue Discord](https://discord.gg/WEu6BdFEtp)
- [bootc Discussion](https://github.com/bootc-dev/bootc/discussions)
- [COSMIC Desktop](https://github.com/pop-os/cosmic-epoch)

## Learn More

- [Universal Blue Documentation](https://universal-blue.org/)
- [bootc Documentation](https://containers.github.io/bootc/)
- [Bluefin Documentation](https://projectbluefin.io)
- [COSMIC Desktop Project](https://github.com/pop-os/cosmic-epoch)

## Security

This template provides security features for production use:
- Optional SBOM generation (Software Bill of Materials) for supply chain transparency
- Optional image signing with cosign for cryptographic verification
- Automated security updates via Renovate
- Build provenance tracking

These security features are disabled by default to allow immediate testing. When you're ready for production, see the "Love Your Image? Let's Go to Production" section above to enable them.

## License

Apache 2.0 - See LICENSE file

---

Based on the [finpilot template](https://github.com/castrojo/finpilot) from [Universal Blue Project](https://universal-blue.org/)
