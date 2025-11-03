using Modding;
using UnityEngine;
using System;
using System.Collections;
using DiscordRPC;
using UnityEngine.SceneManagement;

namespace HollowPresence
{
    public class HollowPresence : Mod
    {
        private DiscordRpcClient client;
        private long startTimeUnix;

        public override string GetVersion() => "1.4";

        public override void Initialize()
        {
            Log("HollowPresence initialized!");

            // Discord client
            client = new DiscordRpcClient("YOUR_DISCORD_APP_CLIENT_ID");
            client.Initialize();

            startTimeUnix = DateTimeOffset.Now.ToUnixTimeSeconds();

            // Start update loop
            GameManager.instance.StartCoroutine(UpdatePresenceRoutine());
        }

        private IEnumerator UpdatePresenceRoutine()
        {
            while (true)
            {
                try { UpdatePresence(); }
                catch (Exception ex) { LogError($"Presence update failed: {ex}"); }

                yield return new WaitForSeconds(8f);
            }
        }

        private void UpdatePresence()
        {
            var gm = GameManager.instance;
            var pd = PlayerData.instance;

            if (gm == null || pd == null) return;

            string scene = SceneManager.GetActiveScene().name;
            string zone = pd.GetString("currentMapZone");
            string details = "";
            string state = zone;

            // Bench detection
            if (pd.atBench)
            {
                details = "Resting at a Bench";
            }
            // Hot spring detection
            else if (scene.ToLower().Contains("hot_spring"))
            {
                details = "Relaxing in a Hot Spring";
            }
            // Stag station detection
            else if (scene.ToLower().Contains("stag"))
            {
                details = "At the Stag Station";
            }
            // Boss detection
            else if (IsBossPresent())
            {
                string bossName = GetCurrentBossName();
                details = $"Fighting {bossName}";
            }
            else
            {
                details = "Exploring";
            }

            // Update Discord Rich Presence
            client.SetPresence(new RichPresence()
            {
                Details = details,
                State = state,
                Assets = new Assets()
                {
                    LargeImageKey = "hollowknight",
                    LargeImageText = "Hollow Knight",
                    SmallImageKey = GetSmallIcon(scene),
                    SmallImageText = scene
                },
                Timestamps = new Timestamps() { StartUnixMilliseconds = startTimeUnix }
            });
        }

        // Dynamically detect if a boss is present in the scene
        private bool IsBossPresent()
        {
            var bosses = GameObject.FindObjectsOfType<HealthManager>();
            foreach (var hm in bosses)
            {
                if (hm.IsBoss())
                    return true;
            }
            return false;
        }

        // Dynamically get the boss’s display name
        private string GetCurrentBossName()
        {
            var bosses = GameObject.FindObjectsOfType<HealthManager>();
            foreach (var hm in bosses)
            {
                if (hm.IsBoss())
                {
                    string name = hm.gameObject.name;

                    // Clean up the name
                    name = name.Replace("(Clone)", "").Replace("_", " ");
                    return System.Globalization.CultureInfo.CurrentCulture.TextInfo.ToTitleCase(name);
                }
            }
            return "Unknown Boss";
        }

        private string GetSmallIcon(string scene)
        {
            scene = scene.ToLower();
            if (scene.Contains("stag")) return "stag";
            if (scene.Contains("bench")) return "bench";
            if (scene.Contains("hot_spring")) return "hotspring";
            if (scene.Contains("dream")) return "dream";
            if (IsBossPresent()) return "boss";
            return "knight";
        }

        public override void Unload()
        {
            if (client != null)
            {
                client.ClearPresence();
                client.Dispose();
                client = null;
            }

            base.Unload();
        }
    }

    // Extension method to check if a HealthManager is a boss
    public static class HealthManagerExtensions
    {
        public static bool IsBoss(this HealthManager hm)
        {
            // Bosses have a 'boss' tag or are in BossManager scenes
            if (hm == null) return false;

            return hm.gameObject.CompareTag("Boss") || hm.name.ToLower().Contains("boss");
        }
    }
}
