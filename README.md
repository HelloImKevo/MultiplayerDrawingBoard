![dotnet-version](https://img.shields.io/badge/.NET-7-purple)

# MultiplayerDrawingBoard
A lightweight proof-of-concept multiplayer drawing board. Uses .NET 7, RxJS, and SignalR.

---

# Dependencies
Download and install .NET:  
https://dotnet.microsoft.com/en-us/download

If you are using Mac Apple Silicon Chip, install .NET 7.0 for Arm64.


### NuGet: Dotnet Entity Framework

https://www.nuget.org/packages/dotnet-ef/  

Run:
```shell
dotnet tool install --global dotnet-ef --version 7.0.4
```

### Git Fundamentals

Configure your Git Access. I recommend using Git Credential Manager Core:  
https://github.com/git-ecosystem/git-credential-manager/blob/release/docs/install.md

Run:
```shell
git credential-manager configure

git config --show-origin --get credential.helper
```

You can erase existing OSX Keychain credentials with:
```shell
git config --unset credential.helper

git credential-osxkeychain erase

[Press Return]
```

See: https://docs.github.com/en/get-started/getting-started-with-git/updating-credentials-from-the-macos-keychain  

If you're running into issues, make sure you aren't using a repo that was originally configured for SSH,
and you are now attempting to use the same repo with Git Credential Manager - this will be significantly 
more complex to configure.

---

# Helpful Quick References

## Dotnet Commands

List currently installed tools:
```shell
dotnet tool list -g
```

## Creating a New Project
Run:
```shell
dotnet --info

dotnet -h
```

Create new solution file and project file:
```shell
dotnet new sln

dotnet new webapi -n API

ls

dotnet sln -h

dotnet sln add API/

dotnet sln list
```

Open Visual Studio Code, do "Show All Commands" (SHIFT + CMD + P), type "PATH", 
select option: **Shell command: Install 'code' command in PATH.**

If you run into a "Permission Denied" error, try uninstalling it first.

Open project by running this in the Terminal:
```shell
code .
```

## Angular App and API Quick Start

In one VS Code Terminal, `cd API/` then run:
```
dotnet watch --no-hot-reload
```

And in another Terminal instance, `cd client` then run:
```
kill -9  $(lsof -t -i:4200);ng serve
ng serve
```

Now both services (API server and client app) will be running. Open this URL in a browser:  
https://localhost:4200/


## Troubleshooting Dotnet Runtime

If you encounter this error:
```
System.IO.IOException: Failed to bind to address https://127.0.0.1:5001: address already in use.
```

You can try running:
```
lsof -i:5001
```

Example output:
```
Google    36572 john   70u  IPv6 0x1223c1a0ae0e2ced  0t0  TCP localhost:60565->localhost:commplex-link (ESTABLISHED)
API       96825 john  244u  IPv4 0x1223c1aa43f9eee5  0t0  TCP localhost:commplex-link (LISTEN)
API       96825 john  245u  IPv6 0x1223c1a0ae0e3bed  0t0  TCP localhost:commplex-link (LISTEN)
API       96825 john  256u  IPv6 0x1223c1a0ae0c436d  0t0  TCP localhost:commplex-link->localhost:60565 (ESTABLISHED)
```

Then run `kill -9 <pid>` to manually kill the leftover "API" processes. In the above example, it 
should be: `kill -9 96825`.
