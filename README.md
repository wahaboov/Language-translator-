// Compile with this command: 
// g++ -o Translator.exe AWahabProject.cpp -std=c++11 -I. -lwinhttp
// -
#include <iostream>
#include <string>
#include <windows.h>
#include <winhttp.h>
#include <codecvt>
#include "json.hpp" 

#pragma comment(lib, "winhttp.lib")

using json = nlohmann::json;
using namespace std;

// Convert UTF-8 to wstring
wstring Utf8ToWstring(const string& utf8) {
    int size_needed = MultiByteToWideChar(CP_UTF8, 0, utf8.c_str(), -1, NULL, 0);
    wstring wideStr(size_needed, 0);
    MultiByteToWideChar(CP_UTF8, 0, utf8.c_str(), -1, &wideStr[0], size_needed);
    return wideStr;
}

// Function to encode URL
wstring UrlEncode(const wstring& input) {
    wstring encoded;
    wchar_t hex[] = L"0123456789ABCDEF";
    for (wchar_t ch : input) {
        if (iswalnum(ch)) {
            encoded += ch;
        } else {
            encoded += L'%';
            encoded += hex[(ch >> 4) & 0xF];
            encoded += hex[ch & 0xF];
        }
    }
    return encoded;
}

// Function to translate text using Google Translate
wstring GoogleTranslate(const wstring& text, const wstring& targetLang) {
    HINTERNET hSession = WinHttpOpen(L"Mozilla/5.0", WINHTTP_ACCESS_TYPE_DEFAULT_PROXY,
                                     WINHTTP_NO_PROXY_NAME, WINHTTP_NO_PROXY_BYPASS, 0);
    if (!hSession) return L"Error: Unable to open session";

    HINTERNET hConnect = WinHttpConnect(hSession, L"translate.googleapis.com", INTERNET_DEFAULT_HTTPS_PORT, 0);
    if (!hConnect) return L"Error: Unable to connect";

    wstring encodedText = UrlEncode(text);
    wstring requestPath = L"/translate_a/single?client=gtx&sl=en&tl=" + targetLang + L"&dt=t&q=" + encodedText;

    HINTERNET hRequest = WinHttpOpenRequest(hConnect, L"GET", requestPath.c_str(), NULL, WINHTTP_NO_REFERER,
                                            WINHTTP_DEFAULT_ACCEPT_TYPES, WINHTTP_FLAG_SECURE);
    if (!hRequest) return L"Error: Unable to open request";

    const wchar_t* headers = L"User-Agent: Mozilla/5.0\r\n";
    if (!WinHttpSendRequest(hRequest, headers, -1, NULL, 0, 0, 0)) {
        return L"Error: Request failed";
    }

    if (!WinHttpReceiveResponse(hRequest, NULL)) {
        return L"Error: Response failed";
    }

    DWORD bytesRead;
    char buffer[4096] = {0};
    string response;

    while (WinHttpReadData(hRequest, buffer, sizeof(buffer), &bytesRead) && bytesRead) {
        response.append(buffer, bytesRead);
    }

    // Cleanup
    WinHttpCloseHandle(hRequest);
    WinHttpCloseHandle(hConnect);
    WinHttpCloseHandle(hSession);

    try {
        json jsonResponse = json::parse(response);
        return Utf8ToWstring(jsonResponse[0][0][0].get<string>());
    } catch (exception& e) {
        return L"Error: Failed to parse response";
    }
}

int main() {
    map<int, pair<wstring, wstring>> languages = {
        {1, {L"French", L"fr"}},
        {2, {L"Spanish", L"es"}},
        {3, {L"Dutch", L"nl"}},
        {4, {L"German", L"de"}}
    };

    wcout << L"Select a language for translation:\n";
    for (const auto& lang : languages) {
        wcout << lang.first << L". " << lang.second.first << L"\n";
    }

    int choice;
    wcout << L"Enter choice (1-4): ";
    wcin >> choice;
    wcin.ignore();

    if (languages.find(choice) == languages.end()) {
        wcout << L"Invalid choice. Exiting...\n";
        return 0;
    }

    wstring targetLangCode = languages[choice].second;
    wstring targetLangName = languages[choice].first;

    wcout << L"Selected Language: " << targetLangName << L"\n";
    wstring input;
    wcout << L"Enter English text (or 'exit' to quit):\n";

    while (true) {
        wcout << L"> ";
        getline(wcin, input);

        if (input == L"exit") break;

        wstring translatedText = GoogleTranslate(input, targetLangCode);
        wcout << targetLangName << L" Translation: " << translatedText << L"\n\n";
    }

    return 0;
}
