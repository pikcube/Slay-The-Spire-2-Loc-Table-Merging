# A Brief Background

Slay the Spire 2 loads its text from a series of json files inside of `localization/{lang}/{file}.json`. Each file consists of a set of key value pairs, and when the game tries to display text, it loads the value for a key specified in the game. The multiple files mainly exist for the purposes of organization, but specifying the language in the file path means Megacrit can change the language by just changing which folder to navigate to in the file path. It also means their translators can just adjust the json files without needing to touch the code or recompile the game.

These files are often referred to as `LocTables`, and they can get pretty long. `cards.json` is over 1200 lines long at the time of writing, and will likely get longer as the game progresses.

When modders create their own cards, they create their own `cards.json` file that is appended to the original at runtime by the game's mod loader. Modders don't have to put all their card's text into `cards.json`, but they are going to need to override or patch much of the game's logic if they want the game to look elsewhere. While stuffing all the card strings into one json file is fine for single character mods, mods with teams working on multiple characters are going to find it difficult to keep track of where all the strings are. Additionally, having multiple people make edits to the same file increases the probability of merge conflicts.

# Loc Table Aliasing

When I came across this problem while working on Dice The Spire, I didn't want to deal with this, so I went about writing some code that let me merge a json file into a table different than its file name. The game already has a defined function that loads in modded text, no reason I can't append some extra code on the end to read in some extra json files according so some schema I developed.

I want to offer a quick explanation of how to set this up because it's not particularly complicated and it massively decreases development headaches.

## Step 1: Define Your Aliases

I opted to load my aliases from a json file, which has the benefit of being easier for an analyzer to find. The syntax here is incredibly simple, the BasePath defines which table we are going to merge into, and the Alias paths define which json files we want to append.

```json
[
  {
    "BasePath": "cards",
    "AliasPaths": [
      "cards.inventor",
      "cards.thief",
      "cards.warrior"
    ]
  },
  {
    "BasePath": "relics",
    "AliasPaths": [
      "relics.inventor",
      "relics.thief",
      "relics.warrior"
    ]
  },
  {
    "BasePath": "potions",
    "AliasPaths": [
      "potions.inventor",
      "potions.thief",
      "potions.warrior"
    ]
  },
  {
    "BasePath": "powers",
    "AliasPaths": [
      "powers.inventor",
      "powers.thief",
      "powers.warrior"
    ]
  }
]
```
I stuck this file inside of `res://{ModID}/locAliases.json`, since I'll need to load it at runtime.

Here's the matching C# class that we deserialize these entries into.

```cs
public record LocAliasInfo(string BasePath, List<string> AliasPaths);
```

##  Step 2: Load Your Alias Definitions Into the Game

We are going to load this in using `System.Text.Json` (since that is what Slay the Spire 2 uses). The function to load them takes the mod id and uses it to derive the path of the json file.

```cs
    public static void LoadJson(string modId)
    {
        string jsonString = FileAccess.GetFileAsString(GetAllFilesRecursive($"res://{modId}").Single(f => f.EndsWith("locAliases.json")));
        LocAliasInfo[] locInfos = JsonSerializer.Deserialize<LocAliasInfo[]>(jsonString) ?? throw new NoNullAllowedException();
        foreach (LocAliasInfo locInfo in locInfos)
        {
            Register(modId, locInfo.BasePath, locInfo.AliasPaths);
        }
    }

    private static IEnumerable<string> GetAllFilesRecursive(string directoryPath)
    {
        using DirAccess dir = DirAccess.Open(directoryPath);
        foreach (string file in dir.GetFiles())
        {
            yield return $"{directoryPath}/{file}";
        }

        foreach (string subDirectory in dir.GetDirectories())
        {
            foreach (string file in GetAllFilesRecursive($"{directoryPath}/{subDirectory}"))
            {
                yield return file;
            }
        }
    }
```

We throw an exception if we find multiple files named `locAliases.json` or if no file is found, so the game can inform us that an error has occurred we need to fix.

We then register each LocTable for use later. The logic is a bit more complicated than just "stick it in a list" due to there potentially being multiple languages.

```cs
    private static List<LocAliasInfo> LocAliases { get; } = [];
    
    public static void Register(string modId, string path, params IEnumerable<string> aliases)
    {
        if (!path.EndsWith(".json"))
        {
            path = $"{path}.json"; //Loc files need to end with .json, so we can add that here with a simple check on the file path.
        }

        DirAccess directory = DirAccess.Open(Path.Join($"res://{modId}", "localization"));

        string[] languages = directory.GetDirectories(); //We check here to see what language files the mod actually defined.

        string[] aliasArray = aliases as string[] ?? [.. aliases];

        foreach (string language in languages)
        {
            string basePath = string.Join('/', "res://localization", language, path);
            IEnumerable<string> aliasPaths = aliasArray //Here we build the full path for each language
                .Select(s =>
                {
                    if (!s.EndsWith(".json"))
                    {
                        s = $"{s}.json";
                    }

                    return string.Join('/', $"res://{modId}", "localization", language, s);
                })
                .Where(s => ResourceLoader.Exists(s)); //And the check that the file exists before trying to read from it

            LocAliasInfo? existing = LocAliases.SingleOrDefault(lai => lai.BasePath == basePath); //If there's an existing LocAliasInfo in the registry, we can just append are aliases to that, otherwise we'll make a fresh one.

            if (existing is not null)
            {
                existing.AliasPaths.AddRange(aliasPaths);
            }
            else
            {
                LocAliases.Add(new LocAliasInfo(basePath, [.. aliasPaths]));
            }
        }
    }
```
## Step 3: Patch the Game to Include Your Additional json Files

Now let's actually patch the game to load these additional json files in

```cs
[HarmonyPatch(typeof(ModManager), nameof(ModManager.GetModdedLocTables))]
public static class LocAliasManager
{
    [HarmonyPostfix]
    internal static IEnumerable<string> MergeAliasesIntoTable(IEnumerable<string> __result, string language, string file)
    {
        string path = string.Join('/', "res://localization", language, file); //This will be identical to our base path inside of the LocAlias we stored earlier.
        foreach (string original in __result)
        {
            yield return original; //First we are going to yield every value the base game already provided
        }
        foreach (LocAliasInfo info in LocAliases.Where(lai => lai.BasePath == path))
        {
            foreach (string alias in info.AliasPaths)
            {
                yield return alias; //Then we'll append our additional json files as additional paths to load. The game will take care of actually deserializing them for us.
            }
        }
    }
```

The whole class comes in at under 100 lines, including the definition of `LocAliasInfo` (which I kept in a separate file for organization purposes)
```cs
[HarmonyPatch(typeof(ModManager), nameof(ModManager.GetModdedLocTables))]
public static class LocAliasManager
{
    private static List<LocAliasInfo> LocAliases { get; } = [];

    public static void Register(string modId, string path, params IEnumerable<string> aliases)
    {
        if (!path.EndsWith(".json"))
        {
            path = $"{path}.json";
        }

        DirAccess directory = DirAccess.Open(Path.Join($"res://{modId}", "localization"));

        string[] languages = directory.GetDirectories();

        string[] aliasArray = aliases as string[] ?? [.. aliases];

        foreach (string language in languages)
        {
            string basePath = string.Join('/', "res://localization", language, path);
            IEnumerable<string> aliasPaths = aliasArray
                .Select(s =>
                {
                    if (!s.EndsWith(".json"))
                    {
                        s = $"{s}.json";
                    }

                    return string.Join('/', $"res://{modId}", "localization", language, s);
                })
                .Where(s => ResourceLoader.Exists(s));

            LocAliasInfo? existing = LocAliases.SingleOrDefault(lai => lai.BasePath == basePath);

            if (existing is not null)
            {
                existing.AliasPaths.AddRange(aliasPaths);
            }
            else
            {
                LocAliases.Add(new LocAliasInfo(basePath, [.. aliasPaths]));
            }
        }
    }

    [HarmonyPostfix]
    internal static IEnumerable<string> MergeAliasesIntoTable(IEnumerable<string> __result, string language, string file)
    {
        string path = string.Join('/', "res://localization", language, file);
        foreach (string original in __result)
        {
            yield return original;
        }
        foreach (LocAliasInfo info in LocAliases.Where(lai => lai.BasePath == path))
        {
            foreach (string alias in info.AliasPaths)
            {
                yield return alias;
            }
        }
    }

    public static void LoadJson(string modId)
    {
        string jsonString = FileAccess.GetFileAsString(GetAllFilesRecursive($"res://{modId}").Single(f => f.EndsWith("locAliases.json")));
        LocAliasInfo[] locInfos = JsonSerializer.Deserialize<LocAliasInfo[]>(jsonString) ?? throw new NoNullAllowedException();
        foreach (LocAliasInfo locInfo in locInfos)
        {
            Register(modId, locInfo.BasePath, locInfo.AliasPaths);
        }
    }

    private static IEnumerable<string> GetAllFilesRecursive(string directoryPath)
    {
        using DirAccess dir = DirAccess.Open(directoryPath);
        foreach (string file in dir.GetFiles())
        {
            yield return $"{directoryPath}/{file}";
        }

        foreach (string subDirectory in dir.GetDirectories())
        {
            foreach (string file in GetAllFilesRecursive($"{directoryPath}/{subDirectory}"))
            {
                yield return file;
            }
        }
    }
}
```

All that is left is to add a call to `LoadJson` to our mod initializer to actually load the json file in.

```cs
LocAliasManager.LoadJson(ModId);
```

# Localization Analyzers

If you are using the templates created by [Alchyr](https://github.com/Alchyr/BaseLib-StS2), then you are likely using their analyzer that detects if loc strings are missing. That analyzer also needs to be modified to be aware of our loc aliases if you don't want any false positives.

The modification is pretty similar to what we did for the base game, we just need to tell the analyzer to read a few extra jsons when starting up analysis

```cs
//Here we read in where the aliases are
AdditionalText? aliasFile = additionalFiles.SingleOrDefault(file => file.Path.EndsWith("locAliases.json"));

Dictionary<string, string> aliases = [];

if (aliasFile is not null)
{
    string jsonString = aliasFile.GetText()?.ToString() ?? "";
    List<LocAliasInfo> aliasInfo = JsonSerializer.Deserialize<List<LocAliasInfo>>(jsonString) ?? [];
    foreach (LocAliasInfo info in aliasInfo)
    {
        foreach (string alias in info.AliasPaths)
        {
            aliases.Add(alias, info.BasePath);
        }
    }
}
```
```cs
// And here we change the table name if it's a registered alias
foreach (string s in locObj.Keys)
{
    if (aliases.TryGetValue(fileKey, out string value))
    {
        _currentLocKeys.Add($"{value}.{s}");
    }
    else
    {
        _currentLocKeys.Add($"{fileKey}.{s}");
    }
}
```

I uploaded a fork of the analyzer to github and nuget with these changes, if anyone else is interested in them.

```xml
<PackageReference Include="Pikcube.Alchyr.Sts2.ModAnalyzers" Version="0.2.9" PrivateAssets="All" />
```
