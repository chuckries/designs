# Source Link Authentication in Visual Studio
*Sometimes source link needs auth and we only support some auth*

## Visual Studio Team Services
Authentication for VSTS is provided by [Signing into Visual Studio](https://msdn.microsoft.com/en-us/library/dn457348.aspx). The Debugger is able to acquire an OAuth token for any signed in AAD or MSA account. This token is included in the Source Link request to any visualstudio.com domain.

## Git Credential Manager For Windows <a id="gcm"></a>
For Source Link requests to non VSTS domains, the Visual Studio leverages the [Git Credential Manager for Windows](https://github.com/Microsoft/Git-Credential-Manager-for-Windows). GCM is used first and foremost by git.exe as a [credential provider](https://git-scm.com/docs/git-credential). VS is able to use GCM as a robust wrapper on top of the [Windows Credential Manager](https://support.microsoft.com/en-us/help/4026814/windows-accessing-credential-manager). GCM also allows VS to leverage credentials that users have already provided to git.exe on the command line. Additionally, GCM supports Two Factor Authentication for services such as GitHub and BitBucket.

### Whitelist
Visual Studio uses a domain whitelist to limit the use of GCM authentication. VS 2017 15.7 ships with only one domain in the whitelist to support GitHub. Editing the whitelist to add custom domains in possible, but VS does not currently have UI to enable editing. 

The whitelist is defined in the Visual Studio private registry under they key `Debugger\GitCredentialManager\Authorities`. Each subkey represents a domain that is whitelisted for GCM authentication. The string value of the subkey is the URL that should be provided to GCM to acquire credentials for authentication.

As an example, we will look at the default GitHub whitelist entry. We inspect the private VS registry using the VSRegEdit.exe tools which is included with Visual Studio

1. Open a Developer Command Prompt for Visual Studio as an Administrator
2. Run the following command and examine the output:
```
C:\Windows\System32>vsregedit.exe read "%VSINSTALLDIR:~0,-1%" HKLM Debugger\GitCredentialManager\Authorities raw.githubusercontent.com string
Name: raw.githubusercontent.com, Value: https://github.com
```

From this we see that the subkey `raw.githubusercontent.com` has the value `https://github.com`. This means Visual Studio will use GCM credentials for `https://github.com` for any Source Link request to the domain `raw.githubusercontent.com`

### Adding to the Whitelist
As a Source Link user, I may want to whitelist my private domain in order to use Source Link inside my company. To do this I must first add my domain to the whitelist. To add to the whitelist in a quick way for testing, I can again use the VSRegEdit.exe command. I will assume that my company's Source Link requests are to `https://privatesrc.mycompany.com`.

1. Open a Developer Command Prompt for Visual Studio as an Administrator
2. Run the following command. 
```
C:\Windows\System32>vsregedit.exe set "%VSINSTALLDIR:~0,-1%" HKLM Debugger\GitCredentialManager\Authorities privatesrc.mycompany.com string https://privatesrc.mycompany.com
```
3. Run `devenv /setup`

Now when a Source Link request is made to `https://privatesrc.mycompany.com`, GCM will be asked to provide credentials. If it cannot find credentials, the user will be presented with a username/password dialog to provide credentials.

### Persisting Whitelist Changes
Changes to the whitelist with VSRegEdit.exe will not persist across updates to Visual Studio. To workaround this, we can make a simple VS extension that will configure the registry. When the extension is installed, it will update the whitelist. As long as the extension is installed, the added domains will not be removed from the whitelist. The following instructions assume Visual Studio 2017 15.7.

1. Ensure that the .NET Desktop Development and Visual Studio Extension Development workloads are installed.
2. File -> New -> Project -> Visual C# -> Extensibility -> VSIX Project: name this SourceLinkAuthorities
3. Project -> Add New Item -> Extensibility -> Visual Studio AsyncPackage: name this SourceLinkAuthoritiesPackage
4. Project -> Add New Item -> General -> Text File: name this Authorities.pkgdef
5. Right click Authorities.pkgdef -> Properties -> set 'Include in VSIX' to True
6. Open Authorities.pkgdef and paste the following. Customize the entries to the domains you want to whitelist:
```
[$RootKey$\Debugger\GitCredentialManager\Authorities]
"privatesrc.mycompany.com"="https://privatesrc.mycompany.com"
"someotherdomain.com"="https://someotherdomain.com"
```
7. Change to the 'Release' configuration and Build the Solution
8. Navigate to the output at `bin\Release\SourceLinkAuthorities.vsix`
9. Double click `SourceLinkAuthorities.vsix` to install the extension to Visual Studio.
10. If you need to change the whitelist, change Authorities.pkgdef, increment the version number in source.extension.vsixmanifest, rebuild, and reinstall the extension.

## GitHub
GitHub authentication is provided automatically using [Git Credential Manager for Windows](#gcm). It is whitelisted by default

## GitHub Enterprise


## BitBucket
BitBucket will be whitelisted by default in Visual Studio 2017 15.8. To enable it in 2017 15.7, use either of the above methods to add "bitbucket.org"="https://bitbucket.org" to the whitelist.

## GitLab
Visual Studio does not currently support authentication for GitLab repositories. GitLab does not support Basic HTTP Authentication for raw file access and Git Credential Manager for Windows does not support GitLab.
- [GitLab: Allow access to raw files using HTTP Basic Auth](https://gitlab.com/gitlab-org/gitlab-ce/issues/36654)
- [GCM: Add support for GitLab and GitLab self-hosted instances](https://github.com/Microsoft/Git-Credential-Manager-for-Windows/issues/480)