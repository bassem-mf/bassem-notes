# Installing Multiple Versions of .NET Side-By-Side on Linux

1. Make sure your system has all the dependencies needed by .NET. The dependencies are listed here: <https://learn.microsoft.com/en-us/dotnet/core/install/linux-scripted-manual#dependencies>

2. Create the DOTNET_ROOT folder. Your user account needs to have write access to this folder because some of the .NET commands will need to write to this folder (for example to install a global dotnet tool).

   ```bash
   mkdir $HOME/.dotnet
   ```

3. Download a version of .NET from <https://dotnet.microsoft.com/en-us/download>. Start with the oldest version you need, newer versions after.

4. Extract the file you downloaded to the DOTNET_ROOT folder.

   ```bash
   tar -zxf <the_file_you_downloaded> -C $HOME/.dotnet
   ```

5. Repeat steps 3 and 4 for all versions you need. Older versions first, newer versions after. Extract them all to the same folder, this won't cause any problems.

6. Update your shell profile to add the DOTNET_ROOT environment variable and append the path to the .NET executables to the PATH environment variable. For Bash, you will edit "~/.bashrc" to add the following lines at the end.

   ```bash
   # dotnet environment variables
   export DOTNET_ROOT=$HOME/.dotnet
   export PATH=$PATH:$DOTNET_ROOT:$DOTNET_ROOT/tools
   ```

7. Restart the terminal then run `dotnet --list-sdks` and `dotnet --list-runtimes` to make sure the expected versions were installed.