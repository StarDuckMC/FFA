package me.papakosmas.ffa;

import com.massivecraft.factions.P;
import java.io.File;
import java.io.PrintStream;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map.Entry;
import java.util.Random;
import java.util.Set;
import java.util.logging.Logger;
import org.bukkit.Bukkit;
import org.bukkit.ChatColor;
import org.bukkit.GameMode;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.Server;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.configuration.file.FileConfiguration;
import org.bukkit.configuration.file.FileConfigurationOptions;
import org.bukkit.entity.Player;
import org.bukkit.event.HandlerList;
import org.bukkit.event.entity.EntityDamageEvent;
import org.bukkit.inventory.Inventory;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.PlayerInventory;
import org.bukkit.plugin.PluginDescriptionFile;
import org.bukkit.plugin.PluginManager;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.potion.PotionEffect;
import org.bukkit.scheduler.BukkitScheduler;

public class FFA extends JavaPlugin
{
  public static FFA plugin;
  public final Logger logger = Logger.getLogger("Minecraft");
  FileConfiguration config;
  Random ran = new Random();
  int tpDelay;
  int increase;
  int startKills;
  int initialAmnt;
  String tpMessage;
  public HashMap<Player, Integer> wait = new HashMap();
  public HashMap<Player, ItemStack[]> saveItems = new HashMap();
  public HashMap<Player, Armor> saveArmor = new HashMap();
  public HashMap<Player, Location> saveLocation = new HashMap();
  public HashMap<Player, Integer> playerKills = new HashMap();
  public HashSet<Player> active = new HashSet();
  public boolean ignore;

  public void onDisable()
  {
    PluginDescriptionFile pdfFile = getDescription();
    this.logger.info(pdfFile.getName() + " Has Been Disabled!");
    saveConfig();
  }

  public void onEnable()
  {
    PluginDescriptionFile pdfFile = getDescription();
    this.logger.info(pdfFile.getName() + ", Version " + 
      pdfFile.getVersion() + ", Has Been Enabled!");
    PluginManager pm = getServer().getPluginManager();
    EntityDamageEvent.getHandlerList().unregister(P.p.entityListener);
    pm.registerEvents(new listener(this), this);
    plugin = this;
    this.config = getConfig();
    handleConfig();
  }

  public void handleConfig() {
    if (!exists()) {
      this.config = getConfig();
      this.config.options().copyDefaults(false);
      this.config.addDefault("Spawn", "1,1,1");
      this.config.addDefault("TPMessage", "§7[§cFFA§7]: Welcome to the FFA Arena!");
      this.config.addDefault("TPDelay", Integer.valueOf(5));
      this.config.addDefault("StartProfitAtKills", Integer.valueOf(4));
      this.config.addDefault("InitialProfit", Integer.valueOf(200));
      this.config.addDefault("KillProfitIncrease", Integer.valueOf(20));
      this.config.options().copyDefaults(true);
      saveConfig();
    }
    readConfig();
  }

  public String Colorize(String s) {
    if (s == null) return null;
    return s.replaceAll("&([0-9a-f])", "§$1");
  }

  public void readConfig() {
    this.initialAmnt = getConfig().getInt("InitialProfit");
    this.startKills = getConfig().getInt("StartProfitAtKills");
    this.increase = getConfig().getInt("KillProfitIncrease");
    this.tpDelay = getConfig().getInt("TPDelay");
    this.tpMessage = Colorize(getConfig().getString("TPMessage"));
  }

  private boolean exists() {
    try {
      File file = new File("plugins/FFA/config.yml");
      return file.exists();
    } catch (Exception e) {
      System.err.println("Error: " + e.getMessage());
    }
    return true;
  }

  public void rTeleport(final Player p)
  {
    int i = Bukkit.getServer().getScheduler().scheduleSyncDelayedTask(this, new Runnable()
    {
      public void run()
      {
        if (FFA.plugin.wait.containsKey(p))
        {
          FFA.plugin.saveItems.put(p, p.getInventory().getContents());
          FFA.plugin.saveArmor.put(p, new Armor(p));
          p.sendMessage(ChatColor.GREEN + 
            "Your inventory has been saved!");
          FFA.plugin.saveLocation.put(p, p.getLocation());
          FFA.plugin.wait.remove(p);
          Location l = null;
          String arenaloc = (String)FFA.this.getConfig().get("Spawn");
          String[] arenalocarr = arenaloc.split(",");
          if (arenalocarr.length == 3)
          {
            int x = Integer.parseInt(arenalocarr[0]);
            int y = Integer.parseInt(arenalocarr[1]);
            int z = Integer.parseInt(arenalocarr[2]);
            l = new Location(FFA.this.getServer().getWorld("world"), x, y, z);
          }
          p.teleport(l);
          p.sendMessage(FFA.this.tpMessage);
          Inventory b = p.getInventory();
          b.clear();
          if (p.hasPermission("ffa.vip")) {
            ItemStack sword = new ItemStack(Material.DIAMOND_SWORD);
            ItemStack food = new ItemStack(
              Material.ROTTEN_FLESH, 16);
            ItemStack axe = new ItemStack(Material.DIAMOND_AXE);
            b.addItem(new ItemStack[] { sword, food, axe });
            p.getInventory().setChestplate(
              new ItemStack(Material.IRON_CHESTPLATE));
            p.getInventory().setLeggings(
              new ItemStack(Material.IRON_LEGGINGS));
            p.getInventory().setHelmet(
              new ItemStack(Material.IRON_HELMET));
            p.getInventory().setBoots(
              new ItemStack(Material.IRON_BOOTS));
          } else {
            ItemStack sword = new ItemStack(Material.IRON_SWORD);
            ItemStack food = new ItemStack(
              Material.ROTTEN_FLESH, 16);
            ItemStack axe = new ItemStack(Material.IRON_AXE);
            b.addItem(new ItemStack[] { sword, food, axe });
            p.getInventory().setChestplate(
              new ItemStack(Material.IRON_CHESTPLATE));
            p.getInventory().setLeggings(
              new ItemStack(Material.IRON_LEGGINGS));
            p.getInventory().setHelmet(
              new ItemStack(Material.IRON_HELMET));
            p.getInventory().setBoots(
              new ItemStack(Material.IRON_BOOTS));
          }
          FFA.plugin.playerKills.put(p, Integer.valueOf(0));
          FFA.this.active.add(p);
        }
      }
    }
    , this.tpDelay * 20);

    plugin.wait.put(p, Integer.valueOf(i));
  }

  public boolean onCommand(CommandSender sender, Command cmd, String commandLabel, String[] args)
  {
    if ((sender instanceof Player))
    {
      Player p = (Player)sender;
      if ((commandLabel.equalsIgnoreCase("ffa")) && 
        (p.hasPermission("FFA.default")))
      {
        if (!this.playerKills.containsKey(p))
        {
          if (args.length == 0)
          {
            p.sendMessage(ChatColor.GREEN + 
              "You are about to enter the arena, please wait " + this.tpDelay + " seconds");
            if (p.getGameMode() == GameMode.CREATIVE)
              p.sendMessage("§7[§cFFA§7]: You joined §cFFA§7 and your gamemode has been switched to Survival!");
            p.setGameMode(GameMode.SURVIVAL);
            removeEffects(p);
            rTeleport(p);
          }
          else if (args.length == 1)
          {
            if ((args[0].equalsIgnoreCase("reload")) && 
              (p.hasPermission("FFA.*"))) {
              reloadConfig();
            }
            else
            {
              int blockX;
              if ((args[0].equalsIgnoreCase("setspawn")) && 
                (p.hasPermission("FFA.*"))) {
                Location loc = p.getLocation();
                blockX = loc.getBlockX();
                int blockY = loc.getBlockY();
                int blockZ = loc.getBlockZ();
                getConfig().set("Spawn", blockX + "," + blockY + "," + blockZ);
                p.sendMessage(ChatColor.GOLD + "FFA Spawn has been set succesfully at: " + blockX + 
                  ", " + blockY + ", " + blockZ);
                getServer().getPluginManager().disablePlugin(this);
                getServer().getPluginManager().enablePlugin(this);
              } else if ((args[0].equalsIgnoreCase("end")) && 
                (p.hasPermission("FFA.*"))) {
                this.ignore = true;
                for (Player ap : this.active) {
                  if (this.saveItems.containsKey(ap))
                  {
                    ap.sendMessage(ChatColor.YELLOW + "FFA has been ended by an admin!");
                    Location l = (Location)this.saveLocation.get(ap);
                    ap.teleport(l);
                    this.saveLocation.remove(ap);

                    Map.Entry[] entries = (Map.Entry[])this.saveItems.entrySet().toArray(new Map.Entry[0]);
                    for (int i = 0; i < this.saveItems.entrySet().size(); i++)
                    {
                      Map.Entry entry = entries[i];

                      Player pl = (Player)entry.getKey();
                      ItemStack[] a = (ItemStack[])entry.getValue();
                      if (pl == ap)
                      {
                        pl.getInventory().clear();
                        pl.getInventory().setContents(a);
                        this.saveItems.remove(pl);
                        this.playerKills.remove(pl);
                        ((Armor)this.saveArmor.get(ap)).equip();
                        pl.sendMessage(ChatColor.GREEN + "Your inventory has been restored!");
                      }
                    }
                  }
                }
                this.active.clear();
                this.ignore = false;
              }
            }
          }
          else
          {
            p.sendMessage(ChatColor.RED + "Invalid command!");
          }
        }
        else
          p.sendMessage(ChatColor.RED + 
            "You can't /ffa while in battle!");
      }
      else {
        p.sendMessage(ChatColor.RED + 
          "You don't have permission for this command!");
      }
    }
    return false;
  }

  public void removeEffects(Player p) {
    for (PotionEffect effect : p.getActivePotionEffects())
      p.removePotionEffect(effect.getType());
  }
}
