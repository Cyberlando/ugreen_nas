## In short:

- Below proceure allows you to read runtime-related data from a UGOS-based NAS directly into Home Assistant.
- It keeps UGOS completely untouched (no additional tools installed on the NAS, so no interference with future updates).
- The process involves two steps:<br/>1) Obtain an individual token for authentication (the rather complicated part).<br/>2) Configure Home Assistant for frequent data polling by utilizing the standard REST integration (simple).
- For step 1, a shell script will take over the major steps and generate your token.

**Important**: All this is still under development and currently optimized for my DXP 4800 Plus.<br/>Different models will require adjustments for volumes/disks; also unit conversions / rounding are not done properly yet, etc.
Bottom line: At this stage, this is a proof-of-concept, currently meant for people who like to tinker.

## Introduction

When I switched from my old QNAP to the new UGreen DXP, I ran into a couple of challenges.

The first issue was migrating my VMs. After nearly 10 years of using that QNAP, I had built up a collection of virtual machines, including my entire Home Automation System with databases spanning the past decade. Initially, I couldn’t get these VMs to a functional state on the new system. After some trial-and-error (and a deeper dive into what was going wrong), I found a relatively simple solution. You can check out the details in [this](https://discord.com/channels/1208438687168335913/1270855790147797104/1318333164455723070) short post on UGreen’s Discord.

The next hurdle was UGOS itself — it’s not exactly chatty when it comes to providing operational data like CPU utilizaion, memory use etc. As QNAP has it's own HA integration, I was used to views like this one:

![image](https://github.com/user-attachments/assets/37f5f5d5-9998-4879-bdfa-8fa4d5590ef0)

Searching the Web for a ready-made HA solution or integration didn't get me any positives (but a lot of useful background information). So I had to start digging myself.

The result can be found below. It is not what I would expect from a 'professional' end-user solution; it isn't even nice code. But it's working.

> If you come across any problems when following this guide, you are welcome to open a thread in the discussions section.<br/>
> If you succeed, please also report back, it's always good to know if things are working.

## Preparations

Before you get started, make sure to gather some important information that you’ll need later. Write it down somewhere — you’ll need it in the upcoming steps:

- The IP address of your NAS (four numbers, e.g., 192.168.178.9).
- The port number your NAS uses for communication (usually 9999).
- The username of a NAS account with administrative privileges.
- The password for that user account.
- A specific number that we’ll extract in step 2 of this guide.

Since you already use most of this information when accessing your NAS through its Web Interface in a browser, it should be easy to find (e.g. the IP address and port number are displayed in your browser’s address bar):

![image](https://github.com/user-attachments/assets/01f2415a-c07f-4730-8150-6131435e11f3)

_Side note: While I initially explored this using a different approach, I will use the Visual Studio Code Server for this guide to make the steps easier to follow. If you haven’t installed it yet as an add-on, now is a good time to do so. Alternatively, you’ll need to manually execute the steps using an SSH shell and transfer files via an SMB connection to Home Assistant, or similar methods._

_Side note 2: All shell commands below can be copied and pasted directly from this guide. After pasting a command, press <Enter> to execute it._

## Step 1: Getting the Token Script Ready
> **Purpose:** Set up the shell script for token generation.

- Open the Visual Studio Code Server, navigate to your file structure on the left, and create a directory named `scripts` (if it doesn’t already exist).
- Inside the `scripts` directory, create a new file called `get_ugreen_token.sh`.
- Copy the content of the `scripts/get_ugreen_token.sh` file from this repository into your newly created file.
- Right-click the file name and select “Open in Integrated Terminal”.
- In the terminal, run the following command to make the script executable: `chmod +x get_ugreen_token.sh`.<br/><br/> ![image](https://github.com/user-attachments/assets/3c4808fb-0aa5-4188-bc4d-96c56c79f3a5)


The script is now ready to use.

## Step 2: Retrieving the Last Piece of Information for the Token
> **Purpose:** Locate the certificate number required for token generation.

- Stay in the terminal window and type: `clear` - this gets us an empty, clean workbench.
- Connect to your NAS via SSH by running: `ssh your_username@your_nas_ip` (example: `ssh tom@192.168.178.9`).
- Enter your password when prompted. You will now see the NAS command prompt (you’re working directly on the NAS).
- Run the following command to list the certificate files: `sudo ls /var/cache/ugreen-rsa` (Note: For security reasons, you will be asked to re-enter your password).
- The output will list two files, e.g., 1000.key and 1000.pub. The number in the file name (e.g., 1000) is the certificate number you need. Write this number down.
- Log off from the NAS SSH session by typing: `exit`.<br/><br/>![image](https://github.com/user-attachments/assets/194275a3-57d7-4f7e-9bee-f43b96ee219c)

You now have the final piece of information required for token generation.

## Step 3: Getting Your Token
> **Purpose:** Generate your authentication token for Home Assistant REST requests.

- Stay in the terminal window, run `clear` again for a clean workbench.
- Run the shell script to generate the token: `./get_ugreen_token.sh` (the `./` at the beginning is important).
- Follow the prompts displayed by the script. You’ll need to provide: The IP address of your NAS, the port number, the username and password, the certificate number retrieved in Step 2.<br/>Note: Password will be asked for again; ignore any _'Could not chdir'_ messages.
- You will be presented with 3 results: (1) an encrypted password, (2) a static token, (3) a session token.
- Select the static token (only this one we need) and copy it to you clipboard. Make sure it is staying there until the end of the next step (safe way is to temporarily paste it somewhere).<br/><br/>![image](https://github.com/user-attachments/assets/e985f25f-0f16-4cfd-a552-08b50d444ef4)


You now have a valid token that can be used to authenticate REST requests from Home Assistant.

## Step 4: Creating Boot-Safe HA Config Entities for Your NAS

> **Purpose:** Ensure your token is easily accessible and quickly adjustable at any time without restarting HA.

The most flexible way to store your token is by using a `text_input` entity. This approach allows you to update the token directly in the Home Assistant developer tools without requiring a restart. Currently, I have no practical experience regarding how frequently a token might be revoked or changed by the NAS, but this method seems like the most logical solution.

Additionally, I prefer to keep configurations organized in separate files. Below, I’ll describe the steps following that principle. However, feel free to adjust the instructions to match your setup or preferences. For example, you could also store the token in `secrets.yaml` if that suits your needs.

Open `configuration.yaml` and add a new package under the `homeassistant` key. Leave the `rest` section commented out for now; we’ll handle that in the next step. As always, pay attention to proper indentation:
```yaml
homeassistant:
  packages:
    ugreen_nas:
      # rest:            !include conf/ugreen_nas_rest.yaml
      text_input:        !include conf/ugreen_nas_text_input.yaml
```


- Create a `conf` directory for your configuration and add a file named `ugreen_nas_text_input.yaml` inside it:![image](https://github.com/user-attachments/assets/c133a6a0-a45f-4b7a-91d2-a81057ecff93)

- Copy the content of the file `conf/ugreen_nas_text_input.yaml` from this repository into the newly created file.
- Restart Home Assistant to apply the changes and create the entities.
- Open **Developer Tools** → **States** in Home Assistant and filter for `ugreen`. For each filtered entity, set your local values, confirm each with 'Set state'.![image](https://github.com/user-attachments/assets/ba4a0f1c-cbb9-4433-8616-c7c266438e5f)


You have now completed the basic configuration and initial setup.

## Step 5: Create Your Entities / Sensors
> **Purpose:** Finalize the setup procedure by defining the sensors to read data from the NAS.


>to be completed by me tomorrow - see conf/ugreen_nas_rest.yaml.
>
>it used to work fine for 3 days until I switched today the REST URL from fixed IP/Port in YAML definition to templated input_text's ...
>
>since then, some sensors are not reporting anymore. manual call in browser with the current token in the same URL gives positive JSON results / dict. (HUH? - to be evaluated - still a proof-of-concept)

![image](https://github.com/user-attachments/assets/72fdf1d3-abaa-4e91-90a6-af98c128836d)

