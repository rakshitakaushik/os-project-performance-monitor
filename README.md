#include <iostream>
#include <chrono>
#include <Windows.h>
#include <iostream>
#include <Pdh.h>
#include <wlanapi.h>
#pragma comment(lib, "wlanapi.lib")
#pragma comment(lib, "pdh.lib")
using namespace std;
using namespace chrono;
class CPU {
private:
    time_point<high_resolution_clock> start_;
    time_point<high_resolution_clock> end_;
    DWORD uptime_;

    unsigned int interval_;
    ULONGLONG total1_, idle1_, total2_, idle2_;
    unsigned int cpuUsage_;

    static bool GetSystemCPUTime(ULONGLONG& totalTime, ULONGLONG& idleTime) {
        FILETIME ftSysIdle, ftSysKernel, ftSysUser;
        if (!GetSystemTimes(&ftSysIdle, &ftSysKernel, &ftSysUser)) {
            return false;
        }

        ULARGE_INTEGER sysKernel{}, sysUser, sysIdle;
        sysKernel.HighPart = ftSysKernel.dwHighDateTime;
        sysKernel.LowPart = ftSysKernel.dwLowDateTime;
        sysUser.HighPart = ftSysUser.dwHighDateTime;
        sysUser.LowPart = ftSysUser.dwLowDateTime;
        sysIdle.HighPart = ftSysIdle.dwHighDateTime;
        sysIdle.LowPart = ftSysIdle.dwLowDateTime;

        totalTime = sysKernel.QuadPart + sysUser.QuadPart;
        idleTime = sysIdle.QuadPart;

        return true;
    }

public:
    CPU() {
        start_ = high_resolution_clock::now();
        uptime_ = GetTickCount64();
    }

    void PerformCpuIntensiveOperation() {
        for (int i = 0; i < 1000000; i++) {
            for (int j = 0; j < 1000; j++) {
                int k = i * j;
            }
        }
        end_ = high_resolution_clock::now();
    }

    double GetCpuSpeedInGhz() const {
        auto duration = duration_cast<microseconds>(end_ - start_).count();
        double cpu_speed = static_cast<double>(duration) / 1000000.0;
        return cpu_speed;
    }

    DWORD GetUptimeInSeconds() const {
        return uptime_ / 1000;
    }

    DWORD GetUptimeInMinutes() const {
        return GetUptimeInSeconds() / 60;
    }

    DWORD GetUptimeInHours() const {
        return GetUptimeInMinutes() / 60;
    }

    DWORD GetUptimeInDays() const {
        return GetUptimeInHours() / 24;
    }

    CPU(unsigned int interval) : interval_(interval) {}

    void StartMonitoring() {
        GetSystemCPUTime(total1_, idle1_);
        Sleep(interval_);
        GetSystemCPUTime(total2_, idle2_);
        LONGLONG total = total2_ - total1_;
        LONGLONG idle = idle2_ - idle1_;
        cpuUsage_ = static_cast<unsigned int>(100 * (total - idle) / total);
    }

    void PrintCPUInfo() const {
        double cpu_speed = GetCpuSpeedInGhz();
        DWORD days = GetUptimeInDays();
        DWORD hours = GetUptimeInHours() % 24;
        DWORD minutes = GetUptimeInMinutes() % 60;
        DWORD seconds = GetUptimeInSeconds() % 60;

        cout << "CPU speed: " << cpu_speed << " GHz" << endl;
        cout << "System uptime: " << days << " days, "
            << hours << " hours, "
            << minutes << " minutes, "
            << seconds << " seconds";
        cout << "\n";
        cout<< "CPU usage: " << cpuUsage_ << "%" << endl;
    }
};

// class Memory for monitoring performance of the CPU
class MemoryUsage
{
    // access specifier private
private:
    // The init() function sets the values of 3 class member variables
    // related to the memory status of a computer system. The code uses the
    // Windows API function GlobalMemoryStatusEx() to get information about available, committed, and
    // total memory. The three variables are assigned these values:
    void init()
    {
        MEMORYSTATUSEX memStatus;
        memStatus.dwLength = sizeof(MEMORYSTATUSEX);
        if (GlobalMemoryStatusEx(&memStatus))
        {
            m_totalMemory = memStatus.ullTotalPhys;   // m_totalMemory: represents the total physical memory in bytes in the system.
            m_availableMemory = memStatus.ullAvailPhys; // m_availableMemory: represents the amount of available RAM in bytes.
            m_committedMemory = memStatus.ullTotalPageFile - memStatus.ullAvailPageFile; //m_committedMemory: represents the total amount of virtual memory that has been committed, but not currently being used.
        }
    }
    unsigned long long m_totalMemory;    // unsigned long long represents positive 64bits
    unsigned long long m_availableMemory;
    unsigned long long m_committedMemory;
    // access specifier public    
public:
    MemoryUsage() // constructor
    {
        init();
    }
    // the below functions will calculate the total_memory , available_memory , used_memory and committed_memory
    double getTotalMemory()
    {
        return (double)(m_totalMemory) / (1024 * 1024 * 1024);
    }
    double getAvailableMemory()
    {
        return (double)(m_availableMemory) / (1024 * 1024 * 1024);
    }
    double getUsedMemory()
    {
        return (double)(m_totalMemory - m_availableMemory) / (1024 * 1024 * 1024);
    }
    double getCommittedMemory()
    {
        return (double)(m_committedMemory) / (1024 * 1024 * 1024);
    }
    void printMemoryUsage() // this function prints the values of the above calculated memories.
    {
        cout << "Total Memory: " << getTotalMemory() << " GB" << endl;
        cout << "Available Memory: " << getAvailableMemory() << " GB" << endl;
        cout << "Used Memory: " << getUsedMemory() << " GB" << endl;
        cout << "Committed Memory: " << getCommittedMemory() << " GB" << endl;
    }
};
class WIFI
{
public:
    float GetWifiSendSpeed()
    {
        PDH_HQUERY query;
        PDH_HCOUNTER counter;
        PDH_FMT_COUNTERVALUE value;
        // Create a query object
        PdhOpenQuery(NULL, 0, &query);
        // Add the network interface send speed counter to the query
        PdhAddCounter(query, L"\\Network Interface(*)\\Bytes Sent/sec", 0, &counter);
        // Collect the data for the counter
        PdhCollectQueryData(query);
        // Wait for a short interval to get accurate data
        Sleep(1000);
        // Collect the data again
        PdhCollectQueryData(query);
        // Get the counter value
        PdhGetFormattedCounterValue(counter, PDH_FMT_DOUBLE, NULL, &value);
        // Close the query
        PdhCloseQuery(query);
        // Convert the Wi-Fi send speed from bytes/sec to kilobits/sec
        float sendSpeedKbps = static_cast<float>(value.doubleValue) * 8 / 1024;
        return sendSpeedKbps;
    }
    float GetWifiReceiveSpeed()
    {
        PDH_HQUERY query;
        PDH_HCOUNTER counter;
        PDH_FMT_COUNTERVALUE value;
        PdhOpenQuery(NULL, 0, &query);
        PdhAddCounter(query, L"\\Network Interface(*)\\Bytes Received/sec", 0, &counter);
        PdhCollectQueryData(query);
        Sleep(1000);
        PdhCollectQueryData(query);
        PdhGetFormattedCounterValue(counter, PDH_FMT_DOUBLE, NULL, &value);
        PdhCloseQuery(query);
        float receiveSpeedKbps = static_cast<float>(value.doubleValue) * 8 / 1024;
        return receiveSpeedKbps;
    }

    void print_WIFI_Speed()
    {
        float sendSpeedKbps = GetWifiSendSpeed();
        float receiveSpeedKbps = GetWifiReceiveSpeed(); // Fixed the error by declaring receiveSpeedKbps variable
        cout << "Wi-Fi Send Speed: " << sendSpeedKbps << " Kbps" << endl;
        cout << "Wi-Fi Receive Speed: " << receiveSpeedKbps << " Kbps" << endl;
    }
    wstring GetWifiSSID() {
        wstring ssid;
        HANDLE handle = nullptr;
        DWORD negotiatedVersion = 0;
        DWORD clientVersion = 2;  // Windows Vista or later
        DWORD result = WlanOpenHandle(clientVersion, nullptr, &negotiatedVersion, &handle);
        if (result != ERROR_SUCCESS)
        {

            cout << "Failed to open Wi-Fi handle. Error code: " << result << endl;
            return ssid;
        }
        PWLAN_INTERFACE_INFO_LIST interfaceList = nullptr;

        result = WlanEnumInterfaces(handle, nullptr, &interfaceList);

        if (result != ERROR_SUCCESS)
        {
            cout << "Failed to enumerate Wi-Fi interfaces. Error code: " << result << endl;
            WlanCloseHandle(handle, nullptr);
            return ssid;
        }

        if (interfaceList->dwNumberOfItems > 0) {
            // Get the first interface

            PWLAN_INTERFACE_INFO interfaceInfo = &(interfaceList->InterfaceInfo[0]);
            // Get the interface GUID

            GUID interfaceGuid = interfaceInfo->InterfaceGuid;
            // Get the WLAN connection attributes
            PVOID connectionAttributes = nullptr;
            DWORD dataSize = 0;




            result = WlanQueryInterface(
                handle,
                &interfaceGuid,
                wlan_intf_opcode_current_connection,
                nullptr,
                &dataSize,
                &connectionAttributes,
                nullptr

            );

            if (result == ERROR_SUCCESS) {

                PWLAN_CONNECTION_ATTRIBUTES connectionAttributesPtr = reinterpret_cast<PWLAN_CONNECTION_ATTRIBUTES>(connectionAttributes);
                for (DWORD i = 0; i < connectionAttributesPtr->wlanAssociationAttributes.dot11Ssid.uSSIDLength; i++) {
                    ssid.push_back(static_cast<wchar_t>(connectionAttributesPtr->wlanAssociationAttributes.dot11Ssid.ucSSID[i]));

                }

            }

            else {

                cout << "Failed to query WLAN interface. Error code: " << result << endl;

            }

            if (connectionAttributes != nullptr) {

                WlanFreeMemory(connectionAttributes);

            }
        }
        else
        {
            cout << "No Wi-Fi interfaces found." << endl;
        }
        if (interfaceList != nullptr) {
            WlanFreeMemory(interfaceList);
        }
        WlanCloseHandle(handle, nullptr);
        return ssid;
    }
    void print_WIFI_SSID()

    {
        wstring ssid = GetWifiSSID();
        if (!ssid.empty())
        {
            wcout << "Wi-Fi SSID: " << ssid << endl;
        }
        else
        {
            cout << "Failed to retrieve Wi-Fi SSID." << endl;
        }
    }
    void print_WIFI()
    {
        print_WIFI_Speed();
        print_WIFI_SSID();
    }
};
int main() {
    cout << endl;
    cout << "--------------------------------------------------" << endl;

    const unsigned int interval = 10;
    CPU cpu(interval);
    cpu.PerformCpuIntensiveOperation();
    cpu.StartMonitoring();
    MemoryUsage memUsage;
    WIFI wifi;
    while (true) {
        int choice;
        cout << "Select the task whose performance you have to monitor: (1) CPU, (2) Memory, (3) WIFI, or (4) Exit: ";
        cin >> choice;

        switch (choice) {
        case 1:
            cout << "Displaying CPU information: \n";
            cpu.PrintCPUInfo();
            break;
        case 2:
            cout << "Displaying Memory information: \n";
            memUsage.printMemoryUsage();
            break;
        case 3:
            cout << "Displaying WIFI information:\n";
            wifi.print_WIFI();
            break;
        case 4:
            cout << "Exiting program...\n";
            return 0;
        default:
            cout << "Invalid input, please try again.\n";
        }
    }

    return 0;
}
