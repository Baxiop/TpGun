If you need help   X: Maxime_On

using { /Verse.org }
using { /Fortnite.com/Devices }
using { /Fortnite.com/Characters }
using { /UnrealEngine.com/Temporary/UI }
using { /UnrealEngine.com/Temporary/SpatialMath }
using { /UnrealEngine.com/Temporary/Diagnostics }
using { /Fortnite.com/Game }
using { /Verse.org/Simulation }
using { /Fortnite.com/UI }
using { /Verse.org/Colors }

# ╔══════════════════════════╗
# ║   Created by :  Baxiop   ║
# ╚══════════════════════════╝

UICountdownTimer := class:
    var TextBlock : text_block = text_block{}
    var Canvas : canvas = canvas{}
    var IsActive : logic = false

    CreateUI() : void =
        set TextBlock = text_block{
            DefaultText := StringToMessage("Cooldown: 0"),
            DefaultTextColor := NamedColors.White,
            DefaultJustification := text_justification.Left,
            DefaultTextSize := 32.0
        }
        set Canvas = canvas{
            Slots := array{
                canvas_slot{
                    Anchors := anchors{Minimum := vector2{X:= 0.5, Y := 1.0}, Maximum := vector2{X:= 0.5, Y := 1.0}},
                    Offsets := margin{Left := -176.0, Top := -137.0, Right := 368.0, Bottom := 41.0},
                    Alignment := vector2{X:= 0.0, Y:=0.0},
                    SizeToContent := true,
                    ZOrder := 0,
                    Widget := TextBlock
                }
            }
        }

    ShowTimer(Player : player): void =
        if (PlayerUI := GetPlayerUI[Player]):
            PlayerUI.AddWidget(Canvas)
            set IsActive = true

    ToFixed2Decimals(Val: float): []char =
        if (IntPart := Floor[Val]):
            IntStr := ToString(IntPart)
            DecVal := (Val - (IntPart * 1.0)) * 100.0
            if (DecPart := Floor[DecVal]):
                DecStr :=
                    if (DecPart < 10):
                        "0" + ToString(DecPart)
                    else:
                        ToString(DecPart)
                return IntStr + "." + DecStr
            else:
                return IntStr + ".00"
        else:
            return "0.00"
 
    UpdateTimer(TimeLeft : float): void =
        if (IsActive = true):
            Formatted := ToFixed2Decimals(TimeLeft)
            TextBlock.SetText(StringToMessage("Cooldown: {Formatted}"))


    HideTimer(Player : player): void =
        if (PlayerUI := GetPlayerUI[Player]):
            PlayerUI.RemoveWidget(Canvas)
            set IsActive = false

    StringToMessage<localizes>(InString : string) : message = "{InString}"

pt_GrabberGun := class(creative_device):

    @editable PlayerSpawner : []player_spawner_device = array{}
    @editable TpGun : conditional_button_device = conditional_button_device{}
    @editable CooldownDuration : float = 5.0

    var IsOnCooldown : logic = false
    var TimerUI : UICountdownTimer = UICountdownTimer{}

    OnBegin<override>()<suspends>: void=
        Print("start script")
        TimerUI.CreateUI()
        for(Spawner : PlayerSpawner):
            Spawner.SpawnedEvent.Subscribe(OnPlayerJoins)

    OnPlayerJoins(Agent: agent): void=
        if(FC := Agent.GetFortCharacter[]):
            FC.DamagedEvent().Subscribe(OnPlayerDamaged)
            FC.DamagedShieldEvent().Subscribe(OnPlayerDamaged)

    OnPlayerDamaged(Result: damage_result): void=
        if (IsOnCooldown = false):
            if:
                Instigator := Result.Instigator?
                InstigatorFC := fort_character[Instigator]
                InstigatorAgent := Instigator.GetInstigatorAgent[]
                DamagedFC := fort_character[Result.Target]
                DamagedAgent := DamagedFC.GetAgent[]
            then:
                if(TpGun.IsHoldingItem[InstigatorAgent]):
                    DamagedTransform := DamagedFC.GetTransform()
                    if (InstigatorFC.TeleportTo[DamagedTransform.Translation, DamagedTransform.Rotation]):
                        set IsOnCooldown = true
                        spawn{CooldownTimer(InstigatorAgent)}
        else:
            Print("Not yet available")

    CooldownTimer(Agent : agent)<suspends> : void =
        if (Player := player[Agent]):
            TimerUI.ShowTimer(Player)
            var TimeLeft : float = CooldownDuration
            loop:
                if (TimeLeft <= 0.0):
                    break
                TimerUI.UpdateTimer(TimeLeft)
                Sleep(0.01)
                set TimeLeft -= 0.01
            TimerUI.HideTimer(Player)
            set IsOnCooldown = false
            Print("Ability is ready!")
