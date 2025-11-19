# Kernel-script
#Easy script 
#!/bin/bash

# ==============================================================================
# Termux Android Kernel Build Automation
# Based on user provided snippets
# ==============================================================================

# Configuration Variables
# EDIT THESE BEFORE RUNNING
KERNEL_REPO="https://github.com/LineageOS/android_kernel_xiaomi_sm8250" # Example: Change this to your device repo
KERNEL_BRANCH="lineage-20" # Change to your branch if needed, or remove --depth=1 below
DEFCONFIG_NAME="vendor_defconfig" # Change to your actual defconfig name

# Colors for output
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m' # No Color

echo -e "${GREEN}Starting Termux Kernel Build Automation...${NC}"

# ==============================================================================
# PHASE 1: CLEANUP
# ==============================================================================
echo -e "${GREEN}[1/3] Performing Cleanup...${NC}"

# Clean Termux packages
pkg clean

# Remove Termux temp files
rm -rf /data/data/com.termux/files/usr/tmp/*

# Clean Proot cache
rm -rf /data/data/com.termux/files/usr/var/lib/proot-distro/installed-rootfs/ubuntu/root/.cache

# Clean Home cache
rm -rf ~/.cache/*

# Clean configs
rm -rf ~/.gitconfig ~/.wget-hsts

# Remove existing Ubuntu container to start fresh
if proot-distro list | grep -q "ubuntu"; then
    echo "Removing existing Ubuntu installation..."
    proot-distro remove ubuntu
fi

# Clean specific Termux data (Safe cleanup)
rm -rf /sdcard/Android/data/com.termux/files/* 2>/dev/null || true

# NetHunter/Root cleanup (Requires permissions, may fail if not root, suppressing errors)
# Note: /data/local usually requires root access.
echo "Attempting to clean NetHunter files (if accessible)..."
rm -rf /data/local/nhsystem/kalifs /data/local/nhsystem/kali 2>/dev/null || true

# Setup storage access
termux-setup-storage

echo -e "${GREEN}Cleanup Complete.${NC}"

# ==============================================================================
# PHASE 2: INSTALLATION & SETUP
# ==============================================================================
echo -e "${GREEN}[2/3] Installing Dependencies...${NC}"

# Install base tools
pkg install git wget proot-distro -y

# Install Ubuntu
echo "Installing Ubuntu Proot..."
proot-distro install ubuntu

# Update Ubuntu and install build dependencies
echo "Setting up Ubuntu build environment..."
proot-distro login ubuntu -- bash -c "
    apt update && \
    apt install -y build-essential bc bison flex libssl-dev git wget zip clang lld
"

# Get Proton Clang
echo "Cloning Proton Clang..."
proot-distro login ubuntu -- bash -c "
    rm -rf /clang  # Ensure clean start
    git clone --depth=1 https://github.com/kdrag0n/proton-clang /clang
"

echo -e "${GREEN}Setup Complete.${NC}"

# ==============================================================================
# PHASE 3: KERNEL BUILD
# ==============================================================================
echo -e "${GREEN}[3/3] Starting Kernel Build...${NC}"

# Clone Kernel and Build
proot-distro login ubuntu -- bash -c "
    # Clean previous directory if exists
    rm -rf kernel

    # Clone the kernel source
    echo 'Cloning Kernel Source...'
    git clone --depth=1 $KERNEL_REPO kernel

    cd kernel

    # Find the defconfig (Listing for user verification)
    echo 'Available defconfigs:'
    ls arch/arm64/configs/*defconfig

    # Export Environment Variables
    export ARCH=arm64
    export PATH=/clang/bin:\$PATH

    # Build commands
    echo 'Building config: $DEFCONFIG_NAME...'
    make O=out $DEFCONFIG_NAME

    echo 'Compiling Kernel (Using all available cores)...'
    make O=out -j\$(nproc) Image.gz-dtb
"

# ==============================================================================
# FINISH
# ==============================================================================
RESULT_PATH="/data/data/com.termux/files/usr/var/lib/proot-distro/installed-rootfs/ubuntu/root/kernel/out/arch/arm64/boot/Image.gz-dtb"

if [ -f "$RESULT_PATH" ]; then
    echo -e "${GREEN}Build Successful!${NC}"
    echo "Kernel located at:"
    echo "$RESULT_PATH"
    
    # Optional: Copy to internal storage for easy access
    cp "$RESULT_PATH" /sdcard/Image.gz-dtb
    echo "Copied to /sdcard/Image.gz-dtb"
else
    echo -e "${RED}Build finished, but Image.gz-dtb was not found. Check for errors above.${NC}"
fi
