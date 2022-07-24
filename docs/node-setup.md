# Node Setup

Since the commands are for Ubuntu, you'll have to modify them if you're using a different OS.

1. Set up the dependencies:
    1. Update the package list:

        ```sh
        sudo apt-get update
        ```
    2. Install any available updates:

        ```sh
        sudo apt upgrade -y
        ```
    3. Install the toolchain, and ensure accurate time synchronization:

        ```sh
        sudo apt-get install make build-essential gcc git jq chrony -y
        ```
    4. Download a version of Go that's greater than or equal to 1.18, and less than 2:

        ```sh
        wget https://golang.org/dl/go1.18.2.linux-amd64.tar.gz
        ```
    5. Install Go:

        ```sh
        sudo tar -C /usr/local -xzf go1.18.2.linux-amd64.tar.gz
        ```
    6. Delete the Go download:

        ```sh
        rm go1.18.2.linux-amd64.tar.gz
        ```
    7. Configure Go:

        ```sh
        tee -a ~/.profile << EOF
        export GOROOT=/usr/local/go
        export GOPATH=$HOME/go
        export GO111MODULE=on
        export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
        EOF
        ```
    8. Refresh environment variables:

        ```sh
        source ~/.profile
        ```
2. Install Juno:
    1. Clone the repo:

        ```sh
        git clone https://github.com/CosmosContracts/juno
        ```
    2. Change the directory:

        ```sh
        cd juno
        ```
    3. Fetch the tags:

        ```sh
        git fetch
        ```
    4. Check out the first version's tag:

        ```sh
        git checkout v3.0.0
        ```
    5. Install:

        ```sh
        make install
        ```
    6. If the installation succeeded, then the following command will print `v3.0.0`:

        ```sh
        junod version
        ```
3. Configure Juno:
    1. Create a variable for the repo:

        ```sh
        CHAIN_REPO="https://raw.githubusercontent.com/CosmosContracts/mainnet/main/juno-1"
        ```
    2. Create a variable for the peers:

        ```sh
        PEERS="$(curl -sL "$CHAIN_REPO/persistent_peers.txt")"
        ```
    3. Initialize the chain:

        ```sh
        junod init <MONIKER> --chain-id juno-1
        ```

       Replace `<MONIKER>` with the node's moniker (e.g., `node`).
    4. Download the genesis file:

        ```sh
        curl https://share.blockpane.com/juno/phoenix/genesis.json > ~/.juno/config/genesis.json
        ```
    5. Set the peers:

        ```sh
        sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" ~/.juno/config/config.toml
        ```
    6. In `~/.juno/config/app.toml`, set the value of the `halt-height` key to `2616300`.
4. Set up the Juno systemd unit:
    1. Create the unit (replace `ubuntu` with your user):

        ```sh
        sudo tee -a /etc/systemd/system/junod.service << EOF
        [Unit]
        Description=Juno Daemon
        After=network-online.target

        [Service]
        User=ubuntu
        ExecStart=/home/ubuntu/go/bin/junod start --x-crisis-skip-assert-invariants --halt-height 2616300
        RestartSec=3
        Restart=no
        LimitNOFILE=4096

        [Install]
        WantedBy=multi-user.target
        EOF
        ```

        - We use the `--x-crisis-skip-assert-invariants` flag to skip a check that takes up to several hours to
          complete. It's recommended to skip it, and it doesn't affect archive nodes anyway.
        - We use the `--halt-height` flag to schedule the node to halt once it needs to be upgraded.
        - We set the `Restart` directive to `no` because we don't want the node to automatically restart after it halts.
          Otherwise the halt would've been useless because the node will resume downloading blocks thereby corrupting
          the DB.
    2. Reload systemd:

        ```sh
        sudo systemctl daemon-reload
        ```
    3. Enable the unit:

        ```sh
        sudo systemctl enable junod
        ```
    4. Start the unit:

        ```sh
        sudo systemctl start junod
        ```
    5. Verify that it's running:

        ```sh
        sudo systemctl status junod
        ```