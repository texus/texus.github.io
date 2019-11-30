---
layout: post
title: Finding physical screen size from HMONITOR using EDID
date: 2019-11-30 10:01
---

It turns out that getting the physical size of a monitor (centimeter or inches instead of pixels) isn't that simple. There are several ways to get the size but not all of them are reliable. Finding out which size belongs to which monitor is also not so easy. In this blog post I will provide a way to find the physical size of a monitor in millimeters and a way to match this to a given HMONITOR.
<!--more-->

Although I will provide a full solution with comments, I will not discuss the different ways that exists to get the screen sizes. For this you can read [this blog post](https://ofekshilon.com/2011/11/13/reading-monitor-physical-dimensions-or-getting-the-edid-the-right-way/), which together with [its update](https://ofekshilon.com/2014/06/19/reading-specific-monitor-dimensions/), provided me with most of the information required to fulfil my task. To match the HMONITOR and HDEVINFO, I however took a different approach. Instead of a partial match between two strings, I found a device id that is identical in both parts of the code, except for the case (one string is a lowercase version).

Disclaimer: although this code has been tested on multiple computers with multiple types of monitors, I do not guarantee that the code will work for all cases.

In order to find the screen sizes, we will use the SetupAPI to extract the [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) block from the registry. Every digital monitor is supposed to provide this EDID and I haven't personally encountered a monitor where the size can't be found using this method. It may however be a good idea to provide an alternative in case there is no EDID, the EDID can't be accessed or if matching the id with HMONITOR ends up not working.

The following helper function will extract the width and height in millimeter for all monitors. It returns the size for each device id.
{% highlight c++ %}
// This function may return more monitors than actually connected to the computer, these will
// be filtered out later when matching the device ids.
typedef std::map<std::wstring, std::pair<int, int> > PhyMonitorSizes; // DeviceId -> (width, height)
PhyMonitorSizes findMonitorSizesFromEDID()
{
    PhyMonitorSizes screenSizes;

    const GUID GUID_DEVINTERFACE_MONITOR = { 0xe6f07b5f, 0xee97, 0x4a90, 0xb0, 0x76, 0x33, 0xf5, 0x7b, 0xf4, 0xea, 0xa7 };
    const HDEVINFO hDevInfo = SetupDiGetClassDevs(&GUID_DEVINTERFACE_MONITOR, NULL, NULL, DIGCF_DEVICEINTERFACE);

    // Instead of creating a buffer in each iteration and calling SetupDiGetDeviceInterfaceDetail twice (once to find the required
    // buffer size and once to actually get the data), we create a buffer up front with the maximum size it can have.
    // The device id that we will match with can be at most 128 characters, so the id can't be any larger here (or we have issues).
    // Note that the buffer is slightly larger than it has to be (as "sizeof(SP_DEVICE_INTERFACE_DETAIL_DATA)"
    // was used instead of "offsetof(SP_DEVICE_INTERFACE_DETAIL_DATA, DevicePath)").
    wchar_t devPathBuffer[sizeof(SP_DEVICE_INTERFACE_DETAIL_DATA_W) + (128 * sizeof(wchar_t))];

    // Loop over the device interfaces using the SetupAPI
    DWORD monitorIndex = 0;
    SP_DEVICE_INTERFACE_DATA devInfo;
    devInfo.cbSize = sizeof(devInfo);
    while (SetupDiEnumDeviceInterfaces(hDevInfo, NULL, &GUID_DEVINTERFACE_MONITOR, monitorIndex, &devInfo))
    {
        ++monitorIndex;

        // Retrieve the id of the device interface
        SP_DEVICE_INTERFACE_DETAIL_DATA_W* devPathData = (SP_DEVICE_INTERFACE_DETAIL_DATA_W*)devPathBuffer;
        devPathData->cbSize = sizeof(SP_DEVICE_INTERFACE_DETAIL_DATA_W);
        SP_DEVINFO_DATA devInfoData;
        memset(&devInfoData, 0, sizeof(devInfoData));
        devInfoData.cbSize = sizeof(devInfoData);
        if (!SetupDiGetDeviceInterfaceDetailW(hDevInfo, &devInfo, devPathData, sizeof(devPathBuffer), NULL, &devInfoData))
            continue; // Error

        // The device id that we found here will be used to match with the HMONITOR later
        const std::wstring deviceId = devPathData->DevicePath;

        // Find the instance id of the device to look up the EDID in the registry
        wchar_t instanceId[MAX_DEVICE_ID_LEN];
        if (!SetupDiGetDeviceInstanceIdW(hDevInfo, &devInfoData, instanceId, MAX_PATH, NULL))
            continue; // Error

        // Find the EDID registry key
        HKEY hEDIDRegKey = SetupDiOpenDevRegKey(hDevInfo, &devInfoData, DICS_FLAG_GLOBAL, 0, DIREG_DEV, KEY_READ);
        if (!hEDIDRegKey || (hEDIDRegKey == INVALID_HANDLE_VALUE))
            continue; // Error

        // Read the EDID data from the registry
        BYTE dataEDID[1024];
        DWORD sizeOfDataEDID = sizeof(dataEDID);
        if (ERROR_SUCCESS == RegQueryValueExW(hEDIDRegKey, L"EDID", NULL, NULL, dataEDID, &sizeOfDataEDID))
        {
            // Extract the width and height of the monitor from the EDID
            int WidthMm = ((dataEDID[68] & 0xF0) << 4) + dataEDID[66];
            int HeightMm = ((dataEDID[68] & 0x0F) << 8) + dataEDID[67];
            screenSizes[deviceId] = std::make_pair(WidthMm, HeightMm);
        }

        RegCloseKey(hEDIDRegKey);
    }

    return screenSizes;
}
{% endhighlight %}

Getting the same device id from the HMONITOR also requires quite some work. The first step is to call `GetMonitorInfo` and get the `szDevice` member from `MONITORINFOEX`, but this only gets us halfway. The device name returned by `GetMonitorInfo` isn't the same as the device id that we got with the SetupAPI, so we will define another helper function that finds a mapping between these device names and ids.
{% highlight c++ %}
typedef std::map<std::wstring, std::wstring> DevNameToDevId; // DeviceName -> DeviceId
DevNameToDevId getDeviceNamesToIdMap()
{
    DevNameToDevId namesToIdMap;

    // Query how many display paths there are
    UINT32 nrPaths;
    UINT32 nrModes;
    GetDisplayConfigBufferSizes(QDC_ONLY_ACTIVE_PATHS, &nrPaths, &nrModes);

    // Retrieve the active display paths.
    // Although we don't need the modes, documentation of QueryDisplayConfig says we can't use NULL for them.
    std::vector<DISPLAYCONFIG_PATH_INFO> paths(nrPaths);
    std::vector<DISPLAYCONFIG_MODE_INFO> modes(nrModes);
    QueryDisplayConfig(QDC_ONLY_ACTIVE_PATHS, &nrPaths, &paths[0], &nrModes, &modes[0], NULL);

    // Loop over the display paths and map the device name to the unique id that we will use
    for (unsigned int i = 0; i < paths.size(); ++i)
    {
        DISPLAYCONFIG_SOURCE_DEVICE_NAME sourceName;
        sourceName.header.type = DISPLAYCONFIG_DEVICE_INFO_GET_SOURCE_NAME;
        sourceName.header.size = sizeof(sourceName);
        sourceName.header.adapterId = paths[i].sourceInfo.adapterId;
        sourceName.header.id = paths[i].sourceInfo.id;
        DisplayConfigGetDeviceInfo(&sourceName.header);

        DISPLAYCONFIG_TARGET_DEVICE_NAME targetName;
        targetName.header.type = DISPLAYCONFIG_DEVICE_INFO_GET_TARGET_NAME;
        targetName.header.size = sizeof(targetName);
        targetName.header.adapterId = paths[i].sourceInfo.adapterId;
        targetName.header.id = paths[i].targetInfo.id;
        DisplayConfigGetDeviceInfo(&targetName.header);

        namesToIdMap[sourceName.viewGdiDeviceName] = targetName.monitorDevicePath;
    }

    return namesToIdMap;
}
{% endhighlight %}

Now it is time to put the pieces together. We get the device name from the HMONITOR, we look it up in the map returned by `getDeviceNamesToIdMap` to get the device id, we look that up in the map from 'findMonitorSizesFromEDID' and then we are done. The only remaining caveat is that the device id that comes from `SetupDiGetDeviceInterfaceDetail` seems to be a lowercase version of the id that we got via `DisplayConfigGetDeviceInfo`, so we must compare the strings as case-insensitive.
{% highlight c++ %}
std::pair<int, int> getMonitorSizeInMillimeter(HMONITOR hMonitor)
{
    const PhyMonitorSizes& sizesById = findMonitorSizesFromEDID();
    const DevNameToDevId& deviceIdsByName = getDeviceNamesToIdMap();

    // Find the device name from the HMONITOR
    MONITORINFOEXW monInfo;
    monInfo.cbSize = sizeof(monInfo);
    if (!GetMonitorInfoW(hMonitor, &monInfo))
        return {}; // Error

    // Find the device id belonging to that name
    auto deviceIdIt = deviceIdsByName.find(monInfo.szDevice);
    if (deviceIdIt == deviceIdsByName.end())
        return {}; // Error

    const std::wstring& deviceId = deviceIdIt->second;

    // We now need to loop over the data returned from FindMonitorSizesFromEDID and do a case-insensitive comparison because
    // the device id from FindMonitorSizesFromEDID seems to be in lowercase.
    for (auto it = sizesById.begin(); it != sizesById.end(); ++it)
    {
        const std::wstring& devId = it->first;
        const std::pair<int, int>& size = it->second;
        if (!caseInsensitiveComparison(deviceId, devId))
            continue;

        // We matched the device id from the HMONITOR with the one from SetupAPI, so we now have its size.
        // Keep in mind that the size is independent of the orientation of the monitor. You may still want
        // to add code that swaps the width and height in case the monitor is rotated (which could easily
        // be detected: if the monitor has a larger width in pixels then it must also have a larger width in mm).
        const int phyWidthMm = size.first;
        const int phyHeightMm = size.second;
        return std::make_pair(phyWidthMm, phyHeightMm);
    }

    // If we pass here then we didn't find a match and we couldn't get the size of this monitor
    return {};
}
{% endhighlight %}

I left out the implementation of `caseInsensitiveComparison` in this blog post, but the full code can be found in [this gist](https://gist.github.com/texus/3212ebc1ed1502ecd265cc7cf1322b02).

