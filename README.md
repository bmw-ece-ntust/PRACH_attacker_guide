# Re-build UE attacker (B210)

<aside>
💡

Reference

[](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md?ref_type=heads)

[](https://hackmd.io/@zhongxin/BJSPWUy90#2-Build-OAI-UE)

</aside>

![image.png](Re-build%20UE%20attacker%20(B210)%2012d1009831438064b6afcf322b4fa252/image.png)

### Install USRP B210 dependency

```bash
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses5 libncurses5-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml
```

### Build UHD from source

```bash
git clone https://github.com/EttusResearch/uhd.git
cd uhd
git checkout v4.6.0.0
cd host
mkdir build
cd build
cmake ../
make -j $(nproc)
make test # This step is optional
sudo make install
sudo ldconfig
```

### Download FPGA Image

```bash
sudo uhd_images_downloader
```

### Install Gnuradio

```bash
sudo apt install gnuradio
```

### Check if the system can recognise B210 through USB

```bash
lsusb
```

![image.png](Re-build%20UE%20attacker%20(B210)%2012d1009831438064b6afcf322b4fa252/image%201.png)

### Test the device with uhd to see if it works

```bash

sudo uhd_find_devices
```

![image.png](Re-build%20UE%20attacker%20(B210)%2012d1009831438064b6afcf322b4fa252/image%202.png)

### Build OAI nrUE for USRP

<aside>

Here have two cases and based on two branch `develop`  and `rework_UE`

[https://github.com/Richard-yq/OAI-UE-MSG3-attacker](https://github.com/Richard-yq/OAI-UE-MSG3-attacker)

[https://github.com/Richard-yq/OAI-UE-MSG1-attacker](https://github.com/Richard-yq/OAI-UE-MSG1-attacker)

</aside>

### Clone and  Checkout to `rework_UE` branch

```bash
git clone https://github.com/Richard-yq/OAI-UE-MSG3-attacker.git
git clone https://github.com/Richard-yq/OAI-UE-MSG1-attacker.git
cd  <your path>
git checkout rework_UE
```

<aside>
💡

### **Clone the Private Repository Using SSH URL**

When cloning a private repository, **always use the SSH URL** instead of HTTPS.

### **Example: Clone using SSH**

```bash

git clone git@github.com:your-username/your-private-repo.git
```

🚫 **Do not use HTTPS (e.g., `https://github.com/...`)** because it requires manually entering your username and password. Using SSH allows automatic authentication.

- How to generate SSH key
    
    ### Generate a New SSH Key
    
    Run the following command to generate an SSH key:
    
    ```bash
    ssh-keygen -t ed25519 -C "your_email@example.com"
    
    ```
    
    🔹 If you are using an older version of OpenSSH that does not support ed25519, use RSA instead:
    
    ```bash
    ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
    
    ```
    
    🔹 After executing the command:
    
    - You will be prompted to enter a file location (press **Enter** to save it as `~/.ssh/id_ed25519` or `~/.ssh/id_rsa`).
    - You will be asked to enter a passphrase (optional but recommended for added security).
    
    ---
    
    ### Add the SSH Key to GitHub
    
    Retrieve your SSH public key by running:
    
    ```bash
    cat ~/.ssh/id_ed25519.pub
    
    ```
    
    (If using RSA, run `cat ~/.ssh/id_rsa.pub` instead.)
    
    - Copy the output, which starts with `ssh-ed25519` or `ssh-rsa`.
    - Open **GitHub** → Click on your profile picture (top right) → **Settings**.
    - In the left menu, go to **SSH and GPG keys** → Click **New SSH key**.
    - Enter a title (e.g., "My Laptop") and paste the copied SSH key into the **Key** field.
    - Click **Add SSH key**.
    
    ---
    
    ### Test the SSH Connection
    
    Run the following command in your terminal:
    
    ```bash
    ssh -T git@github.com
    
    ```
    
    If successful, you should see a message like this:
    
    ```
    Hi your-github-username! You've successfully authenticated, but GitHub does not provide shell access.
    
    ```
    
    This confirms that your SSH key has been successfully added.
    
</aside>

### Install ASN.1

```bash
cd /home/oaiue/oaiue/openairinterface5g/cmake_targets/
./build_oai -I
```

### Install nrscope

```bash
sudo apt install -y libforms-dev libforms-bin
```

### Build the attacker(Same as OAI UE)

```bash
cd /home/oaiue/oaiue/openairinterface5g/cmake_targets/
sudo ./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

## Execute Attacker

```bash
cd openairinterface5g/cmake_targets/ran_build/build
# Without 5G CN
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000  -E --ue-fo-compensation
```

- other case
    
    ```bash
    cd /home/oaiue/oaiue/openairinterface5g/cmake_targets/ran_build/build
    # Without 5G CN
    sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3450720000 --ssb 1518 --sa  -E --ue-fo-compensation
    
    # With 5G CN
    sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ue-fo-compensation --sa -E --uicc0.imsi 001010000000001
    
    sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000  -E --ue-fo-compensation
    
    # -r: bandwidth in terms of RBs (default value 106)
    # -- numerology: numerology index (default value 1)
    # --band: NR band number (default value 78)
    # --C: downlink carrier frequency in Hz (default value 0)
    # --C0: uplink frequency offset for FDD in Hz (default value 0)
    # -s: SSB start subcarrier (default value 512) 
    # --sa: Standalone mode
    # --ue-fo-compensation: enables the frequency offset compenstation at the UE. This is useful when running over the air and/or without an external clock/time source
    ```
    

---

<aside>

MSG1 attacker ✅

![image.png](Re-build%20UE%20attacker%20(B210)%2012d1009831438064b6afcf322b4fa252/image%203.png)

</aside>

<aside>

MSG3 attacker ✅

![image.png](Re-build%20UE%20attacker%20(B210)%2012d1009831438064b6afcf322b4fa252/image%204.png)

</aside>
