```java
package com.simpleplayerprofileinfo;

import me.clip.placeholderapi.PlaceholderAPI;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer;
import org.bukkit.Bukkit;
import org.bukkit.OfflinePlayer;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.command.TabCompleter;
import org.bukkit.entity.Entity;
import org.bukkit.entity.Player;
import org.bukkit.util.RayTraceResult;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.Date;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Locale;
import java.util.Set;

public final class ProfileCommand implements CommandExecutor, TabCompleter {

    private final SimplePlayerProfileInfoPlugin plugin;
    private final LegacyComponentSerializer legacySerializer = LegacyComponentSerializer.legacyAmpersand();

    public ProfileCommand(SimplePlayerProfileInfoPlugin plugin) {
        this.plugin = plugin;
    }

    @Override
    public boolean onCommand(@NotNull CommandSender sender,
                             @NotNull Command command,
                             @NotNull String label,
                             @NotNull String[] args) {

        if (command.getName().equalsIgnoreCase("simpleplayerprofileinfo")) {
            return handlePluginCommand(sender, args);
        }

        return handleProfileCommand(sender, args);
    }

    private boolean handlePluginCommand(CommandSender sender, String[] args) {
        if (args.length == 1 && args[0].equalsIgnoreCase("reload")) {
            handleReload(sender);
            return true;
        }

        sendConfiguredMessage(sender, "messages.plugin-command-usage", "&cUsage: &e/SimplePlayerProfileInfo reload");
        return true;
    }

    private boolean handleProfileCommand(CommandSender sender, String[] args) {
        if (!(sender instanceof Player viewer)) {
            sendConfiguredMessage(sender, "messages.only-players", "&cOnly players can use this command.");
            return true;
        }

        if (!hasUsePermission(viewer)) {
            sendConfiguredMessage(viewer, "messages.no-permission", "&cYou do not have permission to use this command.");
            return true;
        }

        if (args.length > 1) {
            sendConfiguredMessage(viewer, "messages.usage", "&cUsage: &e/profile [playername]");
            return true;
        }

        OfflinePlayer target;

        if (args.length == 1) {
            target = findPlayer(args[0]);

            if (target == null) {
                sendConfiguredMessage(viewer, "messages.player-not-found", "&cPlayer not found.");
                return true;
            }
        } else {
            Player lookedAt = getLookedAtPlayer(viewer);

            if (lookedAt == null) {
                sendConfiguredMessage(
                        viewer,
                        "messages.must-look-or-type",
                        "&cYou must be looking at a player or type &e/profile <playername>&c."
                );
                return true;
            }

            target = lookedAt;
        }

        sendPlayerInfo(viewer, target);
        return true;
    }

    @Override
    public @Nullable List<String> onTabComplete(@NotNull CommandSender sender,
                                                @NotNull Command command,
                                                @NotNull String alias,
                                                @NotNull String[] args) {
        if (args.length != 1) {
            return List.of();
        }

        String input = args[0].toLowerCase(Locale.ROOT);
        Set<String> suggestions = new LinkedHashSet<>();

        if (command.getName().equalsIgnoreCase("simpleplayerprofileinfo")) {
            if (hasReloadPermission(sender)) {
                suggestions.add("reload");
            }
        } else if (hasUsePermission(sender)) {
            for (Player player : Bukkit.getOnlinePlayers()) {
                suggestions.add(player.getName());
            }

            for (OfflinePlayer offlinePlayer : Bukkit.getOfflinePlayers()) {
                String name = offlinePlayer.getName();
                if (name != null && offlinePlayer.hasPlayedBefore()) {
                    suggestions.add(name);
                }
            }
        }

        return suggestions.stream()
                .filter(name -> name.toLowerCase(Locale.ROOT).startsWith(input))
                .sorted(String.CASE_INSENSITIVE_ORDER)
                .limit(50)
                .toList();
    }

    private boolean hasUsePermission(CommandSender sender) {
        return sender.hasPermission("SimplePlayerProfileInfo.use");
    }

    private boolean hasReloadPermission(CommandSender sender) {
        return sender.hasPermission("SimplePlayerProfileInfo.reload");
    }

    private void handleReload(CommandSender sender) {
        if (!hasReloadPermission(sender)) {
            sendConfiguredMessage(sender, "messages.no-permission", "&cYou do not have permission to use this command.");
            return;
        }

        plugin.reloadConfig();
        sendConfiguredMessage(sender, "messages.reload-success", "&aConfiguration reloaded successfully.");
    }

    private void sendPlayerInfo(Player viewer, OfflinePlayer target) {
        List<String> lines = plugin.getConfig().getStringList("info-lines");

        if (lines.isEmpty()) {
            lines = List.of("&ePlayer Username: &f%target_name%");
        }

        for (String line : lines) {
            String parsed = applyPlaceholders(viewer, target, line);
            sendRaw(viewer, parsed);
        }
    }

    private String applyPlaceholders(Player viewer, OfflinePlayer target, String line) {
        String parsed = applyBuiltInPlaceholders(target, line);

        if (isPlaceholderApiEnabled()) {
            parsed = PlaceholderAPI.setPlaceholders(target, parsed);

            Player onlineTarget = target.getPlayer();
            if (onlineTarget != null) {
                parsed = PlaceholderAPI.setRelationalPlaceholders(viewer, onlineTarget, parsed);
            }
        }

        return parsed;
    }

    private String applyBuiltInPlaceholders(OfflinePlayer target, String line) {
        Player onlineTarget = target.getPlayer();
        String unavailable = plugin.getConfig().getString("settings.unavailable-value", "N/A");
        String targetName = target.getName() == null ? "Unknown" : target.getName();

        String targetLevel = unavailable;
        String targetWorld = unavailable;
        String targetHealth = unavailable;
        String targetPing = unavailable;
        String targetGameMode = unavailable;

        if (onlineTarget != null) {
            targetLevel = String.valueOf(onlineTarget.getLevel());
            targetWorld = onlineTarget.getWorld().getName();
            targetHealth = String.format(Locale.US, "%.1f", onlineTarget.getHealth());
            targetPing = String.valueOf(onlineTarget.getPing());
            targetGameMode = onlineTarget.getGameMode().name();
        }

        return line
                .replace("%target_name%", targetName)
                .replace("%target_uuid%", target.getUniqueId().toString())
                .replace("%target_online%", target.isOnline() ? "Online" : "Offline")
                .replace("%target_first_played%", formatTime(target.getFirstPlayed()))
                .replace("%target_last_seen%", formatTime(target.getLastPlayed()))
                .replace("%target_level%", targetLevel)
                .replace("%target_world%", targetWorld)
                .replace("%target_health%", targetHealth)
                .replace("%target_ping%", targetPing)
                .replace("%target_gamemode%", targetGameMode)

                // Aliases matching your example style.
                .replace("%player-name%", targetName)
                .replace("%player-level%", targetLevel);
    }

    private @Nullable Player getLookedAtPlayer(Player viewer) {
        int maxDistance = Math.max(1, plugin.getConfig().getInt("look.max-distance", 20));
        boolean throughBlocks = plugin.getConfig().getBoolean("look.through-blocks", false);

        RayTraceResult result = viewer.rayTraceEntities(maxDistance, throughBlocks);
        if (result == null) {
            return null;
        }

        Entity hitEntity = result.getHitEntity();
        if (!(hitEntity instanceof Player target)) {
            return null;
        }

        if (target.getUniqueId().equals(viewer.getUniqueId())) {
            return null;
        }

        return target;
    }

    private @Nullable OfflinePlayer findPlayer(String name) {
        Player exactOnline = Bukkit.getPlayerExact(name);
        if (exactOnline != null) {
            return exactOnline;
        }

        for (Player onlinePlayer : Bukkit.getOnlinePlayers()) {
            if (onlinePlayer.getName().equalsIgnoreCase(name)) {
                return onlinePlayer;
            }
        }

        OfflinePlayer cached = Bukkit.getOfflinePlayerIfCached(name);
        if (isKnownPlayer(cached)) {
            return cached;
        }

        List<OfflinePlayer> offlinePlayers = new ArrayList<>(List.of(Bukkit.getOfflinePlayers()));
        offlinePlayers.sort(Comparator.comparing(player -> {
            String playerName = player.getName();
            return playerName == null ? "" : playerName.toLowerCase(Locale.ROOT);
        }));

        for (OfflinePlayer offlinePlayer : offlinePlayers) {
            String offlineName = offlinePlayer.getName();
            if (offlineName != null && offlineName.equalsIgnoreCase(name) && offlinePlayer.hasPlayedBefore()) {
                return offlinePlayer;
            }
        }

        return null;
    }

    private boolean isKnownPlayer(@Nullable OfflinePlayer player) {
        return player != null && (player.isOnline() || player.hasPlayedBefore());
    }

    private boolean isPlaceholderApiEnabled() {
        return Bukkit.getPluginManager().isPluginEnabled("PlaceholderAPI");
    }

    private String formatTime(long millis) {
        if (millis <= 0) {
            return "Never";
        }

        String pattern = plugin.getConfig().getString("settings.date-format", "yyyy-MM-dd HH:mm:ss");
        return new SimpleDateFormat(pattern).format(new Date(millis));
    }

    private void sendConfiguredMessage(CommandSender sender, String path, String fallback) {
        String prefix = plugin.getConfig().getString("messages.prefix", "");
        String message = plugin.getConfig().getString(path, fallback);
        sendRaw(sender, prefix + message);
    }

    private void sendRaw(CommandSender sender, String message) {
        Component component = legacySerializer.deserialize(message);
        sender.sendMessage(component);
    }
}
```
