using System;
using System.Collections.Generic;
using System.Windows.Forms;
using GTA;
using GTA.Math;
using GTA.Native;

public class UberLiteV3Plus_Paket1 : Script
{
    private enum JobState
    {
        Idle,
        Cooldown,
        GoingToPickup,
        WaitingForBoarding,
        GoingToDropoff
    }

    private class OfferData
    {
        public Vector3 PickupPos;
        public Vector3 DropoffPos;
        public float Fare;
        public string CustomerLabel;
        public float CustomerRating;
        public string ZoneLabel;
        public float ZoneMultiplier;
        public bool RainSurge;
    }

    private JobState state = JobState.Idle;
    private readonly Random rng = new Random();

    private Ped passenger;
    private Blip pickupBlip;
    private Blip dropoffBlip;

    private Vector3 pickupPos;
    private Vector3 dropoffPos;

    private DateTime cooldownUntil;
    private DateTime pickupAcceptedAt;
    private DateTime waitingForBoardingSince;
    private DateTime boardingCommandCooldown;
    private DateTime nextSpeechAt;

    private float fare = 0f;
    private float tip = 0f;
    private float lastRideTotal = 0f;

    private string customerLabel = "Normal";
    private float customerRating = 5.0f;
    private string zoneLabel = "Standart";
    private bool activeRainSurge = false;

    private float driverRating = 5.0f;
    private int bankBalance = 0;
    private int dailyEarnings = 0;

    private bool policePenaltyApplied = false;
    private bool crashPenaltyApplied = false;
    private bool speedPenaltyApplied = false;
    private bool tipEligible = true;

    private List<OfferData> offerQueue = new List<OfferData>();
    private bool appOpen = false;
    private int selectedOfferIndex = 0;
    private bool showRideSummary = false;
    private DateTime rideSummaryUntil;

    private ScriptSettings settings;

    // Settings
    private float nightMultiplier = 2.0f;
    private int nightStartHour = 22;
    private int nightEndHour = 6;
    private float rainMultiplier = 1.4f;

    private int pickupTimeout = 90;
    private int boardingTimeout = 25;
    private int cancelCooldownSeconds = 30;

    private float baseFareMultiplier = 0.6f;
    private float vipMultiplier = 1.6f;
    private float rushMultiplier = 1.2f;
    private float paranoidMultiplier = 1.3f;
    private float drunkMultiplier = 1.15f;
    private float quietMultiplier = 1.05f;
    private float talkativeMultiplier = 1.10f;

    private float airportMultiplier = 1.35f;
    private float downtownMultiplier = 1.20f;
    private float nightclubMultiplier = 1.25f;

    private readonly Keys openKey = Keys.F6;
    private readonly Keys acceptKey = Keys.Enter;
    private readonly Keys rejectKey = Keys.Back;
    private readonly Keys cancelKey = Keys.Delete;
    private readonly Keys resumeKey = Keys.F7;
    private readonly Keys resetPassengerKey = Keys.F9;
    private readonly Keys upKey = Keys.Up;
    private readonly Keys downKey = Keys.Down;

    public UberLiteV3Plus_Paket1()
    {
        Tick += OnTick;
        KeyDown += OnKeyDown;
        Interval = 100;

        LoadSettings();
        UI.Notify("UberLite Paket1 yuklendi | F6=Uber | F7=Rota | F9=Yolcu reset");
    }

    private void LoadSettings()
    {
        settings = ScriptSettings.Load("scripts\\UberLiteV3Plus_Paket1.ini");

        nightMultiplier = settings.GetValue("NIGHT", "Multiplier", 2.0f);
        nightStartHour = settings.GetValue("NIGHT", "StartHour", 22);
        nightEndHour = settings.GetValue("NIGHT", "EndHour", 6);
        rainMultiplier = settings.GetValue("WEATHER", "RainMultiplier", 1.4f);

        pickupTimeout = settings.GetValue("GENERAL", "PickupTimeout", 90);
        boardingTimeout = settings.GetValue("GENERAL", "BoardingTimeout", 25);
        cancelCooldownSeconds = settings.GetValue("GENERAL", "Cooldown", 30);

        baseFareMultiplier = settings.GetValue("FARE", "BaseMultiplier", 0.6f);

        vipMultiplier = settings.GetValue("AI", "VIPMultiplier", 1.6f);
        rushMultiplier = settings.GetValue("AI", "RushMultiplier", 1.2f);
        paranoidMultiplier = settings.GetValue("AI", "ParanoidMultiplier", 1.3f);
        drunkMultiplier = settings.GetValue("AI", "DrunkMultiplier", 1.15f);
        quietMultiplier = settings.GetValue("AI", "QuietMultiplier", 1.05f);
        talkativeMultiplier = settings.GetValue("AI", "TalkativeMultiplier", 1.10f);

        airportMultiplier = settings.GetValue("ZONE", "AirportMultiplier", 1.35f);
        downtownMultiplier = settings.GetValue("ZONE", "DowntownMultiplier", 1.20f);
        nightclubMultiplier = settings.GetValue("ZONE", "NightclubMultiplier", 1.25f);
    }

    private void OnKeyDown(object sender, KeyEventArgs e)
    {
        if (e.KeyCode == resetPassengerKey)
        {
            ResetPassengerPosition();
            return;
        }

        if (e.KeyCode == cancelKey)
        {
            CancelCurrentJob();
            return;
        }

        if (e.KeyCode == resumeKey)
        {
            ResumeCurrentJob();
            return;
        }

        if (appOpen)
        {
            if (e.KeyCode == upKey)
            {
                if (offerQueue.Count > 0)
                {
                    selectedOfferIndex--;
                    if (selectedOfferIndex < 0) selectedOfferIndex = offerQueue.Count - 1;
                }
                return;
            }

            if (e.KeyCode == downKey)
            {
                if (offerQueue.Count > 0)
                {
                    selectedOfferIndex++;
                    if (selectedOfferIndex >= offerQueue.Count) selectedOfferIndex = 0;
                }
                return;
            }

            if (e.KeyCode == acceptKey)
            {
                AcceptSelectedOffer();
                return;
            }

            if (e.KeyCode == rejectKey)
            {
                appOpen = false;
                UI.Notify("Uber uygulamasi kapandi.");
                return;
            }
        }

        if (state == JobState.Cooldown)
        {
            if (e.KeyCode == openKey)
            {
                int left = Math.Max(0, (int)(cooldownUntil - DateTime.Now).TotalSeconds);
                UI.Notify("Yeni cagri icin bekle: " + left + " sn");
            }
            return;
        }

        if (e.KeyCode == openKey && state == JobState.Idle)
        {
            OpenAppMenu();
        }
    }

    private void OnTick(object sender, EventArgs e)
    {
        if (Game.Player.Character.IsDead)
        {
            UI.Notify("~r~Gorev iptal: oldun.");
            CleanupJob();
            return;
        }

        if (showRideSummary && DateTime.Now < rideSummaryUntil)
        {
            DrawRideSummary();
        }
        else
        {
            showRideSummary = false;
        }

        if (appOpen)
        {
            DrawAppMenu();
            return;
        }

        if (state == JobState.Cooldown)
        {
            int left = Math.Max(0, (int)(cooldownUntil - DateTime.Now).TotalSeconds);
            UI.ShowSubtitle("UBER | Yeni cagri bekleme: " + left + " sn", 1000);

            if (DateTime.Now >= cooldownUntil)
            {
                state = JobState.Idle;
                UI.Notify("Yeni cagrilar tekrar acildi.");
            }
            return;
        }

        if (state == JobState.GoingToPickup)
        {
            if (passenger == null || !passenger.Exists())
            {
                UI.Notify("~r~Musteri kayboldu.");
                StartCooldown();
                CleanupJob();
                return;
            }

            Vehicle veh = Game.Player.Character.CurrentVehicle;
            float dist = Game.Player.Character.Position.DistanceTo(passenger.Position);
            int pickupLeft = Math.Max(0, pickupTimeout - (int)(DateTime.Now - pickupAcceptedAt).TotalSeconds);

            if (veh == null || !veh.Exists())
            {
                UI.ShowSubtitle("UBER | Araca bin ve musteriye git. Kalan: " + pickupLeft + " sn", 1000);
                return;
            }

            if (DateTime.Now > pickupAcceptedAt.AddSeconds(pickupTimeout))
            {
                UI.Notify("~r~Musteri cok bekledi, cagiriyi iptal etti.");
                StartCooldown();
                CleanupJob();
                return;
            }

            if (dist < 10f)
            {
                if (veh.Speed < 0.5f)
                {
                    passenger.Task.EnterVehicle(veh, VehicleSeat.RightRear);
                    waitingForBoardingSince = DateTime.Now;
                    boardingCommandCooldown = DateTime.Now.AddSeconds(2);
                    state = JobState.WaitingForBoarding;
                    UI.ShowSubtitle("UBER | Yolcu biniyor...", 1000);
                }
                else
                {
                    UI.ShowSubtitle("UBER | Yolcunun binmesi icin tamamen dur. Kalan: " + pickupLeft + " sn", 1000);
                }
            }
            else
            {
                UI.ShowSubtitle("UBER | Pickup: " + ((int)dist) + "m | $" + (int)fare + " | " + zoneLabel, 1000);
            }
            return;
        }

        if (state == JobState.WaitingForBoarding)
        {
            if (passenger == null || !passenger.Exists())
            {
                UI.Notify("~r~Yolcu kayboldu.");
                StartCooldown();
                CleanupJob();
                return;
            }

            Vehicle veh = Game.Player.Character.CurrentVehicle;
            if (veh == null || !veh.Exists())
            {
                UI.ShowSubtitle("UBER | Yolcu biniyor, aracta kal.", 1000);
                return;
            }

            int boardLeft = Math.Max(0, boardingTimeout - (int)(DateTime.Now - waitingForBoardingSince).TotalSeconds);

            if (passenger.IsInVehicle(veh))
            {
                UI.Notify("~g~Musteri araca bindi.");
                RemovePickupBlip();
                CreateDropoff();
                state = JobState.GoingToDropoff;
                nextSpeechAt = DateTime.Now.AddSeconds(6);
                return;
            }

            if (DateTime.Now > waitingForBoardingSince.AddSeconds(boardingTimeout))
            {
                UI.Notify("~r~Yolcu cok bekledi ve vazgecti.");
                StartCooldown();
                CleanupJob();
                return;
            }

            if (veh.Speed < 0.5f && DateTime.Now >= boardingCommandCooldown)
            {
                passenger.Task.EnterVehicle(veh, VehicleSeat.RightRear);
                boardingCommandCooldown = DateTime.Now.AddSeconds(2);
            }

            UI.ShowSubtitle("UBER | Yolcunun binmesi bekleniyor... " + boardLeft + " sn", 1000);
            return;
        }

        if (state == JobState.GoingToDropoff)
        {
            float dist = Game.Player.Character.Position.DistanceTo(dropoffPos);
            string surgeText = "";
            if (IsNightTime()) surgeText += " GECE";
            if (activeRainSurge) surgeText += " YAGMUR";

            UI.ShowSubtitle("UBER | " + customerLabel + " | Hedef: " + ((int)dist) + "m | $" + (int)fare + surgeText, 1000);

            HandleCustomerSpeech();
            ApplyRidePenalties();

            if (dist < 15f)
            {
                if (passenger != null && passenger.Exists())
                    passenger.Task.LeaveVehicle();

                tip = CalculateTip();
                lastRideTotal = fare + tip;

                Game.Player.Money += (int)lastRideTotal;
                bankBalance += (int)lastRideTotal;
                dailyEarnings += (int)lastRideTotal;

                if (!policePenaltyApplied && !crashPenaltyApplied && !speedPenaltyApplied)
                {
                    driverRating += 0.1f;
                    if (driverRating > 5.0f) driverRating = 5.0f;
                }

                UI.Notify("~g~Teslim edildi! Ucret: $" + (int)fare + " | Bahsis: $" + (int)tip);
                ShowCustomerFinishText();

                showRideSummary = true;
                rideSummaryUntil = DateTime.Now.AddSeconds(6);

                CleanupJob();
            }
        }
    }

    private OfferData GenerateOffer()
    {
        OfferData offer = new OfferData();

        float pickupDistance = 120f + (float)(rng.NextDouble() * 280f);
        float dropoffDistance = 700f + (float)(rng.NextDouble() * 1800f);

        offer.PickupPos = World.GetNextPositionOnStreet(Game.Player.Character.Position.Around(pickupDistance));
        offer.DropoffPos = World.GetNextPositionOnStreet(offer.PickupPos.Around(dropoffDistance));

        offer.Fare = offer.PickupPos.DistanceTo(offer.DropoffPos) * baseFareMultiplier;

        int type = rng.Next(0, 7);
        if (type == 0)
        {
            offer.CustomerLabel = "Normal";
            offer.CustomerRating = 4.2f;
        }
        else if (type == 1)
        {
            offer.CustomerLabel = "Aceleci";
            offer.CustomerRating = 3.8f;
            offer.Fare *= rushMultiplier;
        }
        else if (type == 2)
        {
            offer.CustomerLabel = "VIP";
            offer.CustomerRating = 4.9f;
            offer.Fare *= vipMultiplier;
        }
        else if (type == 3)
        {
            offer.CustomerLabel = "Paranoyak";
            offer.CustomerRating = 4.4f;
            offer.Fare *= paranoidMultiplier;
        }
        else if (type == 4)
        {
            offer.CustomerLabel = "Sarhos";
            offer.CustomerRating = 3.5f;
            offer.Fare *= drunkMultiplier;
        }
        else if (type == 5)
        {
            offer.CustomerLabel = "Sessiz";
            offer.CustomerRating = 4.6f;
            offer.Fare *= quietMultiplier;
        }
        else
        {
            offer.CustomerLabel = "Konuskan";
            offer.CustomerRating = 4.1f;
            offer.Fare *= talkativeMultiplier;
        }

        offer.ZoneMultiplier = GetZoneMultiplier(offer.PickupPos, out offer.ZoneLabel);
        offer.Fare *= offer.ZoneMultiplier;

        offer.RainSurge = IsRainyWeather();
        if (offer.RainSurge)
            offer.Fare *= rainMultiplier;

        if (IsNightTime())
            offer.Fare *= nightMultiplier;

        return offer;
    }

    private void OpenAppMenu()
    {
        offerQueue.Clear();
        offerQueue.Add(GenerateOffer());
        offerQueue.Add(GenerateOffer());
        offerQueue.Add(GenerateOffer());

        selectedOfferIndex = 0;
        appOpen = true;
        UI.Notify("Uber uygulamasi acildi.");
    }

    private void DrawAppMenu()
    {
        if (offerQueue.Count == 0)
        {
            UI.ShowSubtitle("UBER | Cagri yok", 1000);
            return;
        }

        string text = "UBER PAKET1\n\n";

        for (int i = 0; i < offerQueue.Count; i++)
        {
            OfferData offer = offerQueue[i];
            string prefix = (i == selectedOfferIndex) ? "> " : "  ";
            int pickupMeters = (int)Game.Player.Character.Position.DistanceTo(offer.PickupPos);
            string tag = "";
            if (offer.RainSurge) tag += " YAGMUR";
            if (IsNightTime()) tag += " GECE";

            text += prefix +
                    offer.CustomerLabel +
                    " | $" + (int)offer.Fare +
                    " | * " + offer.CustomerRating.ToString("0.0") +
                    " | " + offer.ZoneLabel +
                    " | " + pickupMeters + "m" + tag + "\n";
        }

        text += "\nYukari/Asagi Sec | Enter Kabul | Backspace Kapat";
        UI.ShowSubtitle(text, 1000);
    }

    private void AcceptSelectedOffer()
    {
        if (offerQueue.Count == 0)
        {
            appOpen = false;
            UI.Notify("Cagri kalmadi.");
            return;
        }

        OfferData offer = offerQueue[selectedOfferIndex];

        pickupPos = offer.PickupPos;
        dropoffPos = offer.DropoffPos;
        fare = offer.Fare;
        customerLabel = offer.CustomerLabel;
        customerRating = offer.CustomerRating;
        zoneLabel = offer.ZoneLabel;
        activeRainSurge = offer.RainSurge;

        offerQueue.RemoveAt(selectedOfferIndex);
        if (selectedOfferIndex >= offerQueue.Count)
            selectedOfferIndex = 0;

        appOpen = false;
        AcceptOffer();
    }

    private void AcceptOffer()
    {
        passenger = World.CreatePed(PedHash.Business01AFM, pickupPos);

        if (passenger == null || !passenger.Exists())
        {
            UI.Notify("~r~Musteri olusturulamadi.");
            state = JobState.Idle;
            return;
        }

        pickupAcceptedAt = DateTime.Now;
        policePenaltyApplied = false;
        crashPenaltyApplied = false;
        speedPenaltyApplied = false;
        tipEligible = true;
        tip = 0f;

        pickupBlip = World.CreateBlip(pickupPos);
        if (pickupBlip != null && pickupBlip.Exists())
        {
            pickupBlip.Color = BlipColor.Blue;
            pickupBlip.ShowRoute = true;
            pickupBlip.Name = "Pickup";
        }

        state = JobState.GoingToPickup;
        ShowCustomerIntro();
    }

    private void CreateDropoff()
    {
        dropoffBlip = World.CreateBlip(dropoffPos);
        if (dropoffBlip != null && dropoffBlip.Exists())
        {
            dropoffBlip.Color = BlipColor.Yellow;
            dropoffBlip.ShowRoute = true;
            dropoffBlip.Name = "Dropoff";
        }
    }

    private void SetupSpeechDelay()
    {
        nextSpeechAt = DateTime.Now.AddSeconds(6 + rng.Next(2, 5));
    }

    private void HandleCustomerSpeech()
    {
        if (DateTime.Now < nextSpeechAt)
            return;

        Vehicle veh = Game.Player.Character.CurrentVehicle;
        if (veh == null || !veh.Exists())
            return;

        float speed = veh.Speed;

        if (customerLabel == "Normal")
        {
            if (speed < 5f) UI.ShowSubtitle("Musteri: Rahat rahat gidiyoruz.", 2000);
            else if (speed > 20f) UI.ShowSubtitle("Musteri: Biraz hizli ama sorun yok.", 2000);
            else UI.ShowSubtitle("Musteri: Guzel, devam.", 2000);
        }
        else if (customerLabel == "Aceleci")
        {
            if (speed < 8f) UI.ShowSubtitle("Musteri: Hadi biraz hizlan!", 2000);
            else UI.ShowSubtitle("Musteri: Aynen boyle, devam et.", 2000);
        }
        else if (customerLabel == "VIP")
        {
            if (speed > 18f) UI.ShowSubtitle("Musteri: Daha sakin sur lutfen.", 2000);
            else UI.ShowSubtitle("Musteri: Konforlu surus, guzel.", 2000);
        }
        else if (customerLabel == "Paranoyak")
        {
            if (speed > 15f) UI.ShowSubtitle("Musteri: Yavas! Polisi ustumuze cekme.", 2000);
            else UI.ShowSubtitle("Musteri: Sessiz sakin git.", 2000);
        }
        else if (customerLabel == "Sarhos")
        {
            UI.ShowSubtitle("Musteri: Hic fark etmez, ucur beni hah!", 2000);
        }
        else if (customerLabel == "Sessiz")
        {
            UI.ShowSubtitle("Musteri: ...", 1200);
        }
        else
        {
            UI.ShowSubtitle("Musteri: Sana bir sey anlatayim mi?", 2000);
        }

        SetupSpeechDelay();
    }

    private void ApplyRidePenalties()
    {
        Vehicle veh = Game.Player.Character.CurrentVehicle;
        if (veh == null || !veh.Exists()) return;

        if (!policePenaltyApplied && Game.Player.WantedLevel > 0)
        {
            driverRating -= 0.3f;
            if (driverRating < 1.0f) driverRating = 1.0f;
            policePenaltyApplied = true;
            tipEligible = false;
            UI.Notify("~r~Polis yuzunden puan dustu! Yeni puan: " + driverRating.ToString("0.0"));
        }

        if (!crashPenaltyApplied && veh.Health < 850)
        {
            driverRating -= 0.2f;
            if (driverRating < 1.0f) driverRating = 1.0f;
            crashPenaltyApplied = true;
            tipEligible = false;
            UI.Notify("~r~Arac hasari yuzunden puan dustu! Yeni puan: " + driverRating.ToString("0.0"));
        }

        if (!speedPenaltyApplied && veh.Speed > 42f)
        {
            driverRating -= 0.1f;
            if (driverRating < 1.0f) driverRating = 1.0f;
            speedPenaltyApplied = true;
            tipEligible = false;
            UI.Notify("~r~Asiri hiz yuzunden puan dustu! Yeni puan: " + driverRating.ToString("0.0"));
        }
    }

    private float CalculateTip()
    {
        if (!tipEligible)
            return 0f;

        float t = 0f;
        if (customerLabel == "VIP") t += fare * 0.12f;
        else if (customerLabel == "Konuskan") t += fare * 0.08f;
        else if (customerLabel == "Sessiz") t += fare * 0.05f;
        else t += fare * 0.06f;

        if (activeRainSurge) t += 10f;
        if (IsNightTime()) t += 8f;
        return t;
    }

    private void ShowCustomerIntro()
    {
        UI.Notify("Musteri: " + customerLabel + " | * " + customerRating.ToString("0.0") + " | " + zoneLabel);
        SetupSpeechDelay();
    }

    private void ShowCustomerFinishText()
    {
        UI.Notify("Yolculuk tamamlandi | " + customerLabel + " | * " + customerRating.ToString("0.0"));
    }

    private void DrawRideSummary()
    {
        string text =
            "YOLCULUK OZETI\n\n" +
            "Ucret: $" + (int)fare + "\n" +
            "Bahsis: $" + (int)tip + "\n" +
            "Toplam: $" + (int)lastRideTotal + "\n" +
            "Banka: $" + bankBalance + "\n" +
            "Bugun: $" + dailyEarnings + "\n" +
            "Surucu Puani: " + driverRating.ToString("0.0");

        UI.ShowSubtitle(text, 1000);
    }

    private bool IsRainyWeather()
    {
        int weatherHash = Function.Call<int>(Hash.GET_PREV_WEATHER_TYPE_HASH_NAME);
        int currentHash = Function.Call<int>(Hash.GET_CURR_WEATHER_STATE, 0, 0, 0);

        // Basit yaklaşım: raining/thunder/clearing gibi durumlarda true
        string weather = Function.Call<string>(Hash.GET_FILENAME_FOR_AUDIO_CONVERSATION, weatherHash);
        if (!string.IsNullOrEmpty(weather))
        {
            weather = weather.ToUpperInvariant();
            if (weather.Contains("RAIN") || weather.Contains("THUNDER") || weather.Contains("CLEARING"))
                return true;
        }
        return false;
    }

    private float GetZoneMultiplier(Vector3 pos, out string label)
    {
        string zone = Function.Call<string>(Hash.GET_NAME_OF_ZONE, pos.X, pos.Y, pos.Z);
        if (string.IsNullOrEmpty(zone))
        {
            label = "Standart";
            return 1.0f;
        }

        zone = zone.ToUpperInvariant();

        if (zone == "AIRP")
        {
            label = "Havaalani";
            return airportMultiplier;
        }

        if (zone == "DOWNT" || zone == "ALTA" || zone == "PILLBX" || zone == "BURTON")
        {
            label = "Sehir Merkezi";
            return downtownMultiplier;
        }

        if (zone == "DELPE" || zone == "DELSOL" || zone == "ROCKF")
        {
            label = "Gece Hayati";
            return nightclubMultiplier;
        }

        label = "Standart";
        return 1.0f;
    }

    private void ResetPassengerPosition()
    {
        if (state != JobState.GoingToPickup && state != JobState.WaitingForBoarding)
        {
            UI.Notify("Resetlenecek yolcu yok.");
            return;
        }

        if (passenger != null && passenger.Exists())
        {
            passenger.Position = pickupPos.Around(3f);
            passenger.Task.StandStill(1000);
            state = JobState.GoingToPickup;
            UI.Notify("Yolcu pickup noktasina geri getirildi.");
        }
    }

    private void CancelCurrentJob()
    {
        if (state == JobState.Idle)
        {
            UI.Notify("Aktif gorev yok.");
            return;
        }

        if (state == JobState.Cooldown)
        {
            UI.Notify("Zaten bekleme suresindesin.");
            return;
        }

        UI.Notify("~r~Gorev iptal edildi.");
        StartCooldown();
        CleanupJob();
    }

    private void ResumeCurrentJob()
    {
        if (state == JobState.Idle)
        {
            UI.Notify("Devam edecek gorev yok.");
            return;
        }

        if (state == JobState.Cooldown)
        {
            int left = Math.Max(0, (int)(cooldownUntil - DateTime.Now).TotalSeconds);
            UI.Notify("Yeni cagri icin bekle: " + left + " sn");
            return;
        }

        if (state == JobState.GoingToPickup || state == JobState.WaitingForBoarding)
        {
            if (pickupBlip != null && pickupBlip.Exists())
            {
                pickupBlip.ShowRoute = true;
                UI.Notify("Pickup rotasi yeniden acildi.");
            }
        }
        else if (state == JobState.GoingToDropoff)
        {
            if (dropoffBlip != null && dropoffBlip.Exists())
            {
                dropoffBlip.ShowRoute = true;
                UI.Notify("Dropoff rotasi yeniden acildi.");
            }
        }
    }

    private void StartCooldown()
    {
        cooldownUntil = DateTime.Now.AddSeconds(cancelCooldownSeconds);
        state = JobState.Cooldown;
    }

    private bool IsNightTime()
    {
        int hour = Function.Call<int>(Hash.GET_CLOCK_HOURS);

        if (nightStartHour > nightEndHour)
            return (hour >= nightStartHour || hour < nightEndHour);
        else
            return (hour >= nightStartHour && hour < nightEndHour);
    }

    private void RemovePickupBlip()
    {
        if (pickupBlip != null && pickupBlip.Exists())
            pickupBlip.Remove();
    }

    private void RemoveDropoffBlip()
    {
        if (dropoffBlip != null && dropoffBlip.Exists())
            dropoffBlip.Remove();
    }

    private void CleanupJob()
    {
        RemovePickupBlip();
        RemoveDropoffBlip();

        if (passenger != null && passenger.Exists())
            passenger.Delete();

        passenger = null;
        customerLabel = "Normal";
        customerRating = 5.0f;
        zoneLabel = "Standart";
        activeRainSurge = false;
        tip = 0f;
        state = (state == JobState.Cooldown) ? JobState.Cooldown : JobState.Idle;
    }
}
