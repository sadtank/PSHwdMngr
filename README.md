# PSHwdMngr
Powershell CLI password manager

* FIPS 140-2 compliant encryption methods
* DB is encrypted on disk and loaded in (.ps1 and .bin db file)
* PSH native (no modules or executable binaries needed)
* TOTP support based on RFC 6238 (Old DBs may need to add a filed for totp support. With db loaded, run:
  
   `$db | Add-Member -NoteProperty "totpSecret" -Value ""; savedb`
* Import from KeePass
* Random password generation with granular complexity
* Diceware password generation with entropy calculation
* Quickly navigate to login url
* Automatic username/pwd copy to clipboard (and clear)

## First use
1. Run the .ps1, importing the functions into the current session. use `. .\PSHwdMngr.ps1` if current working directory and `. "<path>\PSHwdMngr.ps1"` for full path.
2. Ignoring all errors, either create a new entry or `Import-KeePassXML`.
3. Run `lse` to check the contents of `$global:db` in memory.
4. Run `savedb <path> -force $true` to save the DB at the desired directory. (Recommend script directory. Only adjust the $global:autoload variable in the script to match your new db path.)
5. Re-run one-liner from step 1 to clobber functions in memory to load your new auto-load path.
Your db should now auto load every time it's run and prompt for the decryption pwd.

## Info

This script is designed to work in less-permissive environments, where some .net namespaces/methods may not be available. Some `system` namespaces are required for encryption methods, so if `$ExecutionContext.SessionState.LanguageMode` is set to `ConstrainedLanguage` then it may not be possible to run this script. Simalarly, the execution policy (`Get-ExecutionPolicy`) may not allow saved scripts to run, but may allow them to be run from the cmdline (paste all) or run from unsaved ISE.

Encryption relevant functions are grouped in the `Encryption SET` for easy code review reference. The use case for this script includes being able to transfer the db between computers. Many native credential objects entangle the user/computer, meaning the db could not be read on another machine. This instead uses a password as the AES key material, for symmetric encryption.

However, when the db is decrypted it sits in memory under a global variable which is a feature. From there you can access all PSH's cmdline goodness to manipulate the db. However, keep `Auto-Close` enabled and don't run anything else in that PSH window. Be aware that any stdin and stdout may appear in logging, which could be centralized in some environments. Db/key vars are cleared when the db is closed. Best practice is to exit the console and not trust the PSH garbage collector. And obviously, never leave your terminal unattended.

There are three base64 encoded strings. One is for the DiceWare word list and the other two are part of an Easter egg. The paranoid among you will not trust anything I say to assure you they are benign. To inspect them without spoiling the fun, consider what cmdlets are calling the variable(s) holding these encoded strings. They are writing to the host and are not being invoked.

There is no builtin multilane text editor for PSH (or any Win CLI…). So to edit notes the user can choose to use CLI only or popup a form GUI. Check the `Edit-Notes` function (aliased to `vim`) to see the implementation. It’s easy to overwrite the notes field incorrectly, so I recommend using the GUI for multi-line notes whenever possible.

## Tips
- Recommend using `New-DiceWare` to generate a master password for the db. This will be memorable and quick to type. Consider using `New-DiceWare` when needing to generate a secure passphrase which is easy to remember/type (eg, host pwd, password db, etc). Use the random password generator (Get-RandomPassword) for passwords which can be auto typed by PSHwdMngr.
- Every change to your db is a "full send." It's not a bad idea to keep a backup of your db. It's not crazy to want to write down your db pwd somewhere safe... It's good enough for bitcoin millionaires who take custody of their bitcoin, keeping the seed phrase(s) on physical punch cards... which is considered best practice... just saying.
- Run `About-PSHwdMngr` for a list of functions. Most functions have an alias, (`List-Entries` = `lse`, `New-Entry` = `newe`). Most functions accept string and wildcard arguments thusly: `lse sharep*` (sharepoint) will return either a single complete record just like `Get-Entry` (`gete sharep*`) if there is a single match. Or it will return multiple matches if any. Any operation other than read will reject a query returning multiple entries.
- Use a NAS or cloud storage account to sync db updates to multiple machines in near-real time. Autosave will save every change to the db as soon as the change is confirmed. There is one edge case that could be improved. Where two users have the DB open, and both make a change, only the most recent change will survive. This could be fixed by updating the write functions to hold new changes in a record outside the db, each change would not only autosave but would quickly reopen the db, immediately append the temp record into the db, then quickly autosave. In this way the db would drastically reducing the chance of simultaneous edits by multiple users. However, where autoclose is used (default 5 mins), especially where users reopen the db explicitly before making edits, the current implementation is adequate.
- It's handy to dot-walk $db properties and pipe to `clip` if you need to clipboard descriptions, urls, etc... just remember to clear the clipboard for anything sensitive. `closedb` and `done` will do this for you if called.


## Common commands
```
#Load db: will also set $global:path
load-db <fullpath>

#Save db: defaults to $global:path
save-db

#Close db: will use autosave settings for save path
close-db

#Close db and exit the terminal (most secure)
done

#Force save, clobber any existing db at path with new db password
save-db <fullpath> -force $true 

#new entry
newe

#edit entry
edite

#remove entry
#if rm is aliased to Remove-Item, this may be typo prone as it could run against the current working directory...
rme “git h*”

#get entries, accepts wildcard. Use quotes for spaces
lse “git h*”

#use entry
usee “git h*”

#use pwd
usepwd “git h*”

#go to url
useurl “git h*”

#new pwd
Get-RandomPassword

#new diceword generation
New-DiceWare <number of words>
```
## Q/A
Q: Why do urls open in Edge of all things?

A: Feel free to research and find more reliable ways to open default browsers with url arguments from PowerShell... Grep for `microsoft-edge` and replace. It's not straightforward to call other types of browsers from PowerShell, or the default browser... (I'm lazy). Might be a fun weekend project for you.

Q: Why not let functions accept objects from pipeline?

A: This becomes a mess when prioritizing auto-save and auto-close which prevents sensitive vars from existing in memory for too long. As it is, there is a main function called after every command to reset the autosave timer. Other implementations perhaps could use a job, but would not know what the user is doing, and could initiate an autoclose while the user is in the middle of modifying entries. The current implementation would pipeline input would need to skip over the end of a function when output is being passed to another function vie pipeline. This is overly complicated for this implementation. The functionality gained via pipelines would greatly impact security and create much more complexity to design. If your use case needs this functionality, sounds like your next weekend is going to be pretty exciting.

Q: Can I use HIBP?

A: Have I Been Pwned (HIBP) requires downloading and running an executable in order to pull down a complete pwned list (best practice) and there are hundreds of millions of lines to search. Using the API instead costs a little, and I am cheap. It would be totally possible to add a property to the `$db` object, including every record created by the `newe` function, then create a function the checks each pwd in $db using the API or offline list... Sounds like a good weekend project for you...
 
Q: Can I join two DBs? How about split them out?

A: Yes, but you will need to manipulate the underlying $DB array of objects and do that manually. None of the import/load/save functions in PSHwdMngr are built for that specifically. However, creative use of the $DB object and the import/load/save functions may make both possible. Be sure to backup your DB first!

Q: Can I export DBs? What if I want to go back to KeePass...?

A: My master plan is to lock everyone into my system forever. When unencrypted in memory, the $db variable holds everything you need to export as json/csv... sounds like another weekend project for you.
