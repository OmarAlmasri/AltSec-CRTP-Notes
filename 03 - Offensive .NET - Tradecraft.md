# AV bypass – Source Code Obfuscation

Tools such as **Codecepticon** (https://github.com/Accenture/Codecepticon) can also obfuscate the source code to bypass any signature-related detection.
Codecepticon needs to be compiled in Visual Studio and it’s command line generator can help generate an obfuscation command quickly.

Compile the project in Visual Studio and navigate to the output directory, to open the *CommandLineGenerator.html* file.  
Here, you can decide how you want to obfuscate the source code.

![[Pasted image 20251031230156.png]]

You can also use the following command to obfuscate the source code with **Codecepticon**:

```powershell
C:\AD\Tools\Codecepticon.exe --action obfuscate --module csharp --verbose --path "C:\AD\Tools\Rubeus-master\Rubeus.sln" --map-file " C:\AD\Tools\Rubeus-master\Mapping.html" --profile rubeus --rename ncefpavs --rename-method markov --markov-min-length 3 --markov-max-length 10 - markov-min-words 3 --markov-max-words 5 --string-rewrite --string-rewrite method xor
```

# AV bypass - ConfuserEx

A great tool to obfuscate the compiled binary is **ConfuserEx** (https://mkaring.github.io/ConfuserEx/) 
**ConfuserEx** is a free .NET obfuscator, which can stop AVs from performing signature based detection.

