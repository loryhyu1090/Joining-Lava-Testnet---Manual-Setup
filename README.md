# Joining-Lava-Testnet---Manual-Setup
This guide walks you through the manual installation of the node and joining the Lava Testnet. Note that this process does not include the "Cosmovisor" tool, so after installing the initial genesis binary, you will need to upgrade your node incrementally as described in the upgrade instructions.

### **Prerequisites**

1. **Verify hardware requirements are met.**
2. **Install package dependencies.**  
   Note: You may need to run commands as `sudo`.

### **Required Packages Installation**

1. Update package lists and install necessary packages:
    ```bash
    sudo apt update
    sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
    ```

2. Create a temporary directory for the installation:
    ```bash
    temp_folder=$(mktemp -d) && cd $temp_folder
    ```

### **Go Installation**

1. Download and install Go:
    ```bash
    go_package_url="https://go.dev/dl/go1.20.5.linux-amd64.tar.gz"
    go_package_file_name=${go_package_url##*/}

    # Download GO
    wget -q $go_package_url

    # Unpack the GO installation file
    sudo tar -C /usr/local -xzf $go_package_file_name

    # Adjust environment variables
    echo "export PATH=\$PATH:/usr/local/go/bin" >>~/.profile
    echo "export PATH=\$PATH:\$(go env GOPATH)/bin" >>~/.profile
    source ~/.profile
    ```

2. **Verify the Installation:**
   - To check the installed Go version, run:
     ```bash
     go version
     ```
   - Ensure `GOPATH` includes `$HOME/go` by running:
     ```bash
     go env GOPATH
     ```
     If not, add it:
     ```bash
     export GOPATH=$HOME/go
     ```
   - Confirm that `PATH` includes `$HOME/go/bin`:
     ```bash
     echo $PATH
     ```

### **1. Set Up a Local Node**

1. **Download App Configurations:**
    ```bash
    # Clone the configuration repository
    git clone https://github.com/lavanet/lava-config.git
    cd lava-config/testnet-2

    # Load the setup configuration
    source setup_config/setup_config.sh
    ```

2. **Set App Configurations:**
    - Copy the default `lavad` configuration files to the Lava config folder:
      ```bash
      echo "Lava config file path: $lava_config_folder"
      mkdir -p $lavad_home_folder
      mkdir -p $lava_config_folder
      cp default_lavad_config_files/* $lava_config_folder
      ```

3. **Set the Genesis File:**
    - Copy the `genesis.json` file to the Lava config folder:
      ```bash
      cp genesis_json/genesis.json $lava_config_folder/genesis.json
      ```

### **2. Join the Lava Testnet**

1. **Copy the Genesis Binary:**
    - Set the `lavad` binary location and download the genesis binary:
      ```bash
      lavad_binary_path="$HOME/go/bin/"
      mkdir -p $lavad_binary_path

      # Download the genesis binary to the Lava path
      wget -O ./lavad "https://github.com/lavanet/lava/releases/download/v0.21.1.2/lavad-v0.21.1.2-linux-amd64"
      chmod +x lavad

      # Make the binary accessible
      sudo cp lavad /usr/local/bin

      # Verify the installation
      lavad --help
      ```

2. **Create a Systemd Service to Run the Lava Node:**
    - Create the systemd unit file with logrotate:
      ```bash
      echo "[Unit]
      Description=Lava Node
      After=network-online.target

      [Service]
      User=$USER
      ExecStart=$(which lavad) start --home=$lavad_home_folder --p2p.seeds $seed_node
      Restart=always
      RestartSec=180
      LimitNOFILE=infinity
      LimitNPROC=infinity

      [Install]
      WantedBy=multi-user.target" >lavad.service
      sudo mv lavad.service /lib/systemd/system/lavad.service
      ```

3. **Start the Lava Node Service:**
    - Configure the `lavad` service to run on boot and start it:
      ```bash
      sudo systemctl daemon-reload
      sudo systemctl enable lavad.service
      sudo systemctl restart systemd-journald
      sudo systemctl start lavad
      ```

4. **Check the Service Status:**
    - Verify the status of the `lavad` service:
      ```bash
      sudo systemctl status lavad
      ```

    - View the service logs:
      ```bash
      sudo journalctl -u lavad -f
      ```

### **3. Upgrades**

When a new upgrade is required, you will encounter an error indicating the need to upgrade your `lavad` binary. Follow these steps to manually upgrade your node:

1. **Upgrade Configurations:**
    ```bash
    temp_folder=$(mktemp -d) && cd $temp_folder
    required_upgrade_name="v0.21.1.2" # CHANGE THIS
    upgrade_binary_url="https://github.com/lavanet/lava/releases/download/$required_upgrade_name/lavad-$required_upgrade_name-linux-amd64"
    ```

2. **Stop the Current `lavad` Process:**
    ```bash
    source ~/.profile
    sudo systemctl stop lavad
    ```

3. **Download and Replace the Current `lavad` Binary:**
    ```bash
    wget "$upgrade_binary_url" -q -O $temp_folder/lavad
    chmod +x $temp_folder/lavad

    # Replace the old binary with the upgraded one
    sudo cp $temp_folder/lavad $(which lavad)
    ```

4. **Restart the `lavad` Service:**
    ```bash
    sudo systemctl start lavad
    ```

5. **Verify the Node is Syncing:**
    ```bash
    lavad status | jq .SyncInfo.catching_up
    sudo journalctl -u lavad -f
    ```

### **Welcome to the Lava Testnet 🌋**
