# Minecraft-GUI
import org.bukkit.Bukkit;
import org.bukkit.ChatColor;
import org.bukkit.Material;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.inventory.InventoryClickEvent;
import org.bukkit.event.player.PlayerInteractEvent;
import org.bukkit.inventory.Inventory;
import org.bukkit.inventory.ItemStack;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.scheduler.BukkitRunnable;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class ComplexPlugin extends JavaPlugin implements CommandExecutor, Listener {

private final Map<UUID, MuteData> muteData = new HashMap<>();
private final Map<UUID, Integer> cooldowns = new HashMap<>();

@Override
public void onEnable() {
    // Register the command executor and event listener
    getCommand("mute").setExecutor(this);
    getCommand("unmute").setExecutor(this);
    getServer().getPluginManager().registerEvents(this, this);

    // Start a task that runs every minute to update the scoreboard
    new BukkitRunnable() {
        @Override
        public void run() {
            for (Player player : Bukkit.getOnlinePlayers()) {
                updateScoreboard(player);
            }
        }
    }.runTaskTimer(this, 0, 20 * 60);

    // Start a task that runs every 10 seconds to remove expired mutes
    new BukkitRunnable() {
        @Override
        public void run() {
            for (Map.Entry<UUID, MuteData> entry : muteData.entrySet()) {
                if (entry.getValue().isExpired()) {
                    Player target = Bukkit.getPlayer(entry.getKey());
                    if (target != null) {
                        target.sendMessage(ChatColor.GREEN + "You have been unmuted!");
                    }
                    muteData.remove(entry.getKey());
                }
            }
        }
    }.runTaskTimer(this, 0, 20 * 10);
}

@Override
public void onDisable() {
    // Save the mute data to a history file or database
    // ...
}

@Override
public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
    if (!(sender instanceof Player)) {
        sender.sendMessage(ChatColor.RED + "Only players can use this command!");
        return true;
    }

    Player player = (Player) sender;

    if (command.getName().equalsIgnoreCase("mute")) {
        if (!player.hasPermission("complexplugin.mute")) {
            player.sendMessage(ChatColor.RED + "You do not have permission to use this command!");
            return true;
        }

        if (args.length < 2) {
            player.sendMessage(ChatColor.RED + "Usage: /mute <player> <time>");
            return true;
        }

        Player target = Bukkit.getPlayer(args[0]);

        if (target == null) {
            player.sendMessage(ChatColor.RED + "Player not found!");
            return true;
        }

        long duration = parseDuration(args[1]);

        if (duration == -1) {
            player.sendMessage(ChatColor.RED + "Invalid time format! Use '10m', '2h', or '3d'.");
            return true;
        }

        // Mute the player and store the mute data
        target.sendMessage(ChatColor.RED + "You have been muted for " + format
