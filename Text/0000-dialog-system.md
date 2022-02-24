- Feature Name: `dialog_system`
- Start Date: 2022-02-23
- RFC PR: [OpenNefia/Planning#0000](https://github.com/OpenNefia/Planning/pull/0000)
- OpenNefia Issue: [OpenNefia/OpenNefia#0000](https://github.com/OpenNefia/OpenNefia/issues/0000)

# Summary
[summary]: #summary

This will be an implementation of Elona's dialog system in a moddable fashion.

# Motivation
[motivation]: #motivation

Dialog in Elona generally falls into one of three categories:

- Dialog spoken by unique NPCs, which is specific to that character only.
- Dialog spoken by normal NPCs, which can have new choices added from various sources depending on the character's job or other circumstances.
- Dialog spoken during story cutscenes.

Modders should have the option of taking advantage of the "normal NPC" choice generation if they just want to add something like an option to trade options to all NPCs. They should also be able to build their own dialog system that doesn't depend on any of the above paradigms.

Some other concerns:

- A common complaint with previous versions of Elona is that the dialog options for each character are largely the same, with a lack of dynamic options or other sources of variety.
- In keeping with OpenNefia's philosophy of "vanilla as base, and further enhancements later", I personally envision the ability to replace vanilla's UI for selecting dialog options with a different system that might look more VN-like (with standing sprites, and so on). The implementation should not be constrained to vanilla's UI limitations, such as the maximum number of choices that can fit within the dialog window's boundaries.
- If I remember correctly, certain variants of HSP Elona supported multiple portrait variations for a single character. Combined with the above, this should naturally extend to dialog system extensions like standing sprites with the usage of metadata.

Finally, some implementation concerns:

- Adding new dialog should not require setting up a C# development environment, unless new scripted events are needed.
- Dialogs in different languages might need a different structure for each line of text, depending on the needs of the translation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Basic

Adding a dialog tree to a character consists of the following steps:

1. Defining the dialog prototype and its node structure in YAML.
2. Adding the nodes' translated text and script callbacks in Lua for each localization the dialog will support.
3. Adding a `DialogComponent` to the character's `Chara` prototype in YAML, and setting its `id` to your newly created dialog's prototype ID.

First, define a new `Dialog` prototype.

```yaml
# MyMod/Resources/Prototypes/Dialog.yml
- type: Elona.Dialog
  id: MyMod.Test
  startNode: Start
  nodes:
  - !type:TextNode
    id: Start
    choices:
    # '@' means to use the dialog protoype's root locale namespace
    # This expands to 'OpenNefia.Prototypes.Elona.Dialog.MyMod.Test.Start.Choices.NeverMind'
    - target: Okay
      key: @Choices.Okay
    - target: NoThanks # key becomes '@Choices.NoThanks'

  - !type: TextNode
    id: Okay
    choices:
    - target: Start

  - !type: TextNode
    id: NoThanks
    choices:
    - target: __END__ # Ends the dialog.
      key: Elona.Dialog.Common.Bye
```

Next, in the locale namespace of the `Dialog` prototype, add the text you want to display for each node.

```lua
-- MyMod/Resources/Locale/en_US/Prototypes/Dialog.lua
OpenNefia.Prototypes.Elona.Dialog.MyMod.Test =
{
    Start = {
        "Putitoro is kind of expensive, but it tastes okay.",
        "It gives me a guilty conscience, though.",
        "Should I make reservations?",
        
        Choices = {
            Okay = "Okay.",
            NoThanks = "No thanks."
        }
    },

    Okay = {
        "Then it's settled!"
    },
    
    NoThanks = {
        "Oh. That's a shame."
    },
}
```

Finally, add a `Dialog` to the appropriate `Chara` prototype:

```yaml
- type: Entity
  id: MyMod.TestChara
  parent: BaseChara
  components:
  - type: Dialog
    canTalk: true
    dialogID: MyMod.Test
  # <...>
```

## Advanced

### Dialog Callbacks

These are reusable pieces of logic that can be called both from within the dialog tree and inline from locale files. They are defined in C#:

```csharp
public class SetFlagCallback : IDialogCallback 
{
    [Dependency] private readonly ISidequestSystem _sidequests = default!;
    
    [DataField]
    public PrototypeId<SidequestId> Id { get; set; } = default!;
    
    [DataField]
    public int Flag { get; set; } = 0;

    void IDialogCallback.OnTriggered(IDialogContext context)
    {
        _sidequests.SetFlag(Id, Flag);
    }
}
```

Usage in YAML:

```yaml
- type: Elona.Dialog
  id: MyMod.Test
  startNode: Start
  nodes:
  - !type:TextNode
    id: Start
    choices:
    - target: __END__
      
    # Triggered before choices are displayed.
    onStart: !type:PlaySoundCallback
      id: Elona.Offer2

    # Triggered after a choice is selected.
    onFinish: !type:SetFlagCallback
      id: TestSidequest
      flag: 1000
```

Usage in Lua:

```lua
OpenNefia.Prototypes.Elona.Dialog.MyMod.Test =
{
    Start = {
        "Some dialog text.",
        { type = "PlaySound", id = "Elona.Heal1" },
        "More dialog text.",
        { 
            type = "SetFlagCallback", 
            id = "TestSidequest", 
            flag = "200" 
        },
        "Even more dialog text.",
    }
}
```

### Dialog Objects

If the dialog needs one-off, inline logic, you can define a C# class to be used as a *dialog object* that contains extra data and logic local to the dialog.

```yaml
- type: Elona.Dialog
  id: MyMod.Test
  startNode: Start

  # You can instantiate a C# object with the dialog containing one-off logic 
  # that can't be easily reused in other IDialogCallback instances.
  dialogObject: !type:TestDialogObject
    putitCount: 3

  nodes:
  - !type:TextNode
    id: Start
    choices:
    - target: __END__
      key: OpenNefia.Dialog.Common.More
      
    # This would call a method named 'SpawnPutits' inside the TestDialogObject class.
    onStart: !type:DialogBoundCallback
      id: SpawnPutits
```

```csharp
public class TestDialogObject : IDialogObject 
{
    [Dependency] private readonly IEntityGen _gen = default!;
    
    [DataField]
    public int PutitCount { get; set; } = 1
    
    [DialogCallback]
    public void SpawnPutits(IDialogContext context) 
    {
        for (int i = 0; i < PutitCount; i++) 
        {
            _gen.SpawnEntity(Chara.Putit, FindOpenPos());
        }
    }
}
```

### String Interpolation in Dialog Text

Sometimes it's necessary to interpolate text inline in a dialog file, like for displaying prices. In this case, you can replace the strings defined in locale files with functions that return strings. Each function is passed the dialog context, so you can access its variables.

```lua
local CommonLib = require("Elona.CommonLib")

OpenNefia.Prototypes.Elona.Dialog.MyMod.Test =
{
    Start = {
        function(ctxt) 
            return ("You there, %s. The price of your freedom is %d photographs of Lady Jure.")
                :format(CommonLib.GetPlayerName(), ctxt.obj:GetJurePhotoCost())
        end,
        
        Choices = {
            Pay = function(ctxt)
                return ("Pay. (Cost: %d photographs of Jure)"):format(ctxt.obj:GetJurePhotoCost())
            end,
            Refuse = "Refuse."
        }
    }
}
```

Functions useable in the locale environment's Lua context can be defined in one of two ways:

- As part of a dialog function library, accessible via `require()`.

```csharp
[DialogLibrary("Elona.CommonLib")]
public class DialogCommonLib : IDialogLibrary 
{
    [Dependency] private readonly IGameSession _gameSession = default!;
    [Dependency] private readonly IDisplayNameSystem _displayNames = default!;

    [DialogLibFunction]
    public string GetPlayerName() 
    {
        return _displayNames.GetDisplayName(_gameSession.Player);
    }
}
```

- On a dialog object, accessible via `ctxt.obj`.

```csharp
public class TestDialogObject : IDialogObject 
{
    [DialogFunction]
    public int GetJurePhotoCost() => 3;
}
```

### Custom Node Types

You can define new dialog node types in C#. The node type can implement special branching logic, or generate an entirely new node dynamically (for example, to support dynamic choice lists).

```yaml
- type: Elona.Dialog
  id: MyMod.Test
  startNode: Start
  nodes:
  - !type:CheckForItemNode
    id: Start
    itemId: Elona.BlueCapsuleDrug
    onFound: HaveItem
    onNotFound: NoItem
    
  - !type:TextNode
    id: HaveItem
    choices:
    - target: __END__
  - !type:TextNode
    id: NoItem
    choices:
    - target: __END__
```

```csharp
public class CheckForItemNode : IDialogNode 
{
    [Dependency] private readonly IGameSession _gameSession = default!;
    [Dependency] private readonly IInventorySystem _invSys = default!;
    
    [DataField]
    public PrototypeId<EntityPrototype> ItemId { get; set; } = default!;
    
    [DataField]
    public DialogNodeId OnFound { get; set; } = default!;
    
    [DataField]
    public DialogNodeId OnNotFound { get; set; } = default!;

    void IDialogNode.OnReached(IDialogContext context)
    {
        var player = _gameSession.Player;
        
        if (_invSys.HasItem(player, ItemId))
        {
            context.SetNextNode(OnFound);
        }
        else 
        {
            context.SetNextNode(OnNotFound);
        }
    }
}
```

### External Node References

You can reference dialog nodes that exist outside the current dialog. This is used for cases like character roles adding new dialog choices (buying/selling, etc).

```yaml
- type: Elona.Dialog
  id: MyMod.Test
  startNode: Start
  nodes:
  - !type:JumpToNode
    id: Start
    # References the `Elona.Merchant` Dialog prototype.
    target: Elona.Merchant:Buy
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A standalone dialog can be shown by passing the appropriate data to the active dialog UI implementation.

```csharp
var args = new DialogWindow.Args() 
{
    Nodes = _dialogSys.GetNodes(Protos.Dialog.Lomias),
    Speaker = FindChara(Protos.Chara.Lomias)
};
var result = _uiMgr.Query<DialogWindow, DialogWindow.Args, DialogResult>(args);
```

The use of `Nodes` in the arguments allows the dialog to be dynamically generated at runtime.

There would be a `DialogComponent` that stores this data on the entity and triggers the dialog window when they are bumped into by the player.

Ideally, there would also be a way to interact with the dialog engine independently of the UI.

```csharp
var engine = new DialogEngine(Protos.Dialog.Lomias, FindChara(Protos.Chara.Lomias));
engine.StepToNextChoice(); // Advances to next the choice, or stops if the end is reached.

while (!engine.IsFinished) {
    foreach (var choice in engine.CurrentChoices) {
        Console.WriteLine($"{choice.Index}: {choice.Text}");
    }
    engine.SetChoice(0);
    engine.StepToNextChoice();
}
```

## Dialog Callbacks in Lua

Because dialog callbacks can be instantiated from Lua, there will need to be some way of marshalling Lua table data into compatible YAML data.

```csharp
LuaTable table = GetLuaCallbackData(_luaContext);
IDialogCallback callback = _luaToYaml.Convert<IDialogCallback>(table);
callback.OnTriggered(context);
```

# Drawbacks
[drawbacks]: #drawbacks

No real drawbacks, since this is a core vanilla feature.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Languages

There are three languages that can be as part of this system: YAML, Lua and optionally C#. Why not remove the C# requirement, in order to let modders script dialog logic without a compiler?

- It fragments the modding system across the Lua and C# boundary. 
- Modders wanting to reuse dialog components may have to reimplement things in C# if they are defined solely in Lua. 
- C#'s type system improves the robustness of the final code.
- C# is necessary to interface with YAML at a certain level, so letting modders use it is better for ergonomics. It's also possible to have external mods that provide new dialog system types so that modders can reuse them without needing to compile anything.

Using Lua for the locale environment was intended as a compromise to allow for string manipulation and common dialog functions. As with later versions of C++, the way Lua is used is intended to be a small subset that looks more like a data declaration language than for actual scripting.

The convention proposed here is that Lua functions should only be used for purposes of string interpolation, and everything else should be handled by dialog callbacks in C#. There is ultimately nothing stopping modders from breaking those conventions, but this implementation leaves enough flexibility so that vanilla's dialog can be ported over in full.

## Features

Each feature of this system is required in order to address a specific need that arises from porting vanilla's dialog system:

- *Dialog callbacks*: Vanilla's dialog logic contains dozens of instances where arbitrary logic can be run upon entering a dialog node, such as updating sidequest progress. Some of this logic is shared across multiple different dialog definitions, so it would make sense to encapsulate these with classes. With enough of these callbacks defined, as well as composite callbacks, it would be possible to implement complicated dialog logic entirely in YAML without the use of compiled code, which is very convenient for modders.
- *Dialog objects*: Other parts of the dialog logic are one-off and cannot be reused. As such, it makes sense to put these functions in a place specific to that dialog only. Since functions cannot be declared in YAML, the next best place would be C#, as it is fully typed.
- *Custom dialog nodes*: Sometimes vanilla's dialog logic will use custom branching that can't be represented as a simple list of choices. Examples include the check for blue capsule drags with the \<Kaneda Bike\>'s dialog.
- *External node references*: They're necessary to implement character roles, which can add new dialog options to NPCs.
- *Dialog callbacks in Lua*: Vanilla's dialog logic sometimes puts arbitary logic between the lines of text of a large node. The only way around this would be to split up each node at every point a callback is used, but this makes the overall flow of the dialog graph confusing to follow since the structure of the Lua code depends on the logical structure of the YAML.
- *Lua dialog libraries*: The text is defined in Lua, so there has to be a way of getting data from the C# runtime into Lua. The only practical way to do this is by using functions.

# Prior art
[prior-art]: #prior-art

- ON/Lua has [its own dialog system](https://github.com/Ruin0x11/OpenNefia/blob/develop/src/mod/elona_sys/dialog/api/Dialog.lua), although the amount of magic is uses owing to dynamic typing is significant, leading to a lack of maintainability. 
  + Its [node format](https://github.com/Ruin0x11/OpenNefia/blob/develop/src/mod/elona/data/dialog/unique/lomias.lua) opts for putting the locale keys inline with the rest of the definition, instead of having text lines of text intuited based on the number of entries in the array-like part of the locale table. I found this too verbose, and it also forced all translations to have the same number of lines for each node, even if that were impractical.
  + The system of defining logic inline as callbacks worked well at the time, but this was only practical because of dynamic typing and the definition system being part of a Turing-complete language instead of a data serialization format like YAML.
  + There were several different ways of defining nodes based on how the data was shaped. One example is the ability to have `text` fields defined as single strings or functions that returned strings instead of just lists. This was changed in this proposal to reduce confusion and provide just a fwe blessed ways of defining things.
- C:DDA has a dialog system implemented in JSON, but the node types they offer are hardcoded.
- [Ink](https://www.inklestudios.com/ink/) is a specialized language for this purpose. However, its implementation is too general for a project with specific needs like OpenNefia, and it uses its own bespoke syntax.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What will the `require()` path namespacing look like for Lua dialog libraries?

# Future possibilities
[future-possibilities]: #future-possibilities

- Dialog objects could be extended such that you can retrieve their state from within locale files, for use with string interpolation.
- Dialog text in locale files could be "annotated" with extra dialog callbacks for setting portrait/standing sprite variations. If a compatible UI mod is not installed, those callbacks would be ignored, and vanilla's behavior would be used instead.
- Dialog callbacks could be strung together with a `CompositeNode` type, which runs a list of child `IDialogCallback`s in order. This could replicate much of the dialog logic that was previously defined as Lua callbacks in ON/Lua.
