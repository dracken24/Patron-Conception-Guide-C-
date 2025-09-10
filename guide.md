# Guide des Patrons de Conception

## Table des matières

1. [Partie 1 : Problèmes et solutions](#partie-1--problemes-et-solutions)
   - [1.1 Gestion des ressources partagées](#11-gestion-des-ressources-partagees)
   - [1.2 Notification d'événements](#12-notification-devenements)
   - [1.3 Création d'objets complexes](#13-creation-dobjets-complexes)
   - [1.4 Création de familles d'objets](#14-creation-de-familles-dobjets)
   - [1.5 Intégration d'interfaces incompatibles](#15-integration-dinterfaces-incompatibles)
   - [1.6 Extension de fonctionnalités](#16-extension-de-fonctionnalites)
   - [1.7 Simplification d'interfaces complexes](#17-simplification-dinterfaces-complexes)
   - [1.8 Traitement d'objets composites](#18-traitement-dobjets-composites)
   - [1.9 Choix d'algorithmes](#19-choix-dalgorithmes)
   - [1.10 Gestion d'états](#110-gestion-detats)

2. [Partie 2 : Similitudes, différences et risques de confusion](#partie-2--similitudes-differences-et-risques-de-confusion)
   - [2.1 Strategy vs State](#21-strategy-vs-state)
   - [2.2 Décorateur vs Façade vs Adaptateur](#22-decorateur-vs-facade-vs-adaptateur)
   - [2.3 Composite vs Décorateur](#23-composite-vs-decorateur)
   - [2.4 Factory vs Singleton](#24-factory-vs-singleton)

---

## Partie 1 : Problèmes et solutions

### 1.1 Gestion des ressources partagees

**Problème :** Vous avez besoin d'une seule instance d'une classe pour gérer une ressource partagée dans un jeu vidéo (gestionnaire de sauvegarde, système audio, gestionnaire de ressources).

**Solution :** Utilisez le patron **Singleton** pour garantir qu'une seule instance de la classe existe dans toute l'application.

**Exemple de code :**

```csharp
public class GameSaveManager
{
    private static GameSaveManager _instance;
    private static readonly object _lock = new object();
    private bool _initialized = false;
    
    private Dictionary<string, object> saveData;
    private int currentSlot;
    
    private GameSaveManager() { }
    
    public static GameSaveManager Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = new GameSaveManager();
                    }
                }
            }
            return _instance;
        }
    }
    
    public void Initialize()
    {
        if (!_initialized)
        {
            saveData = new Dictionary<string, object>();
            currentSlot = 1;
            _initialized = true;
        }
    }
    
    public void SaveGame(Dictionary<string, object> playerData, int level, int score)
    {
        string saveKey = $"slot_{currentSlot}";
        Dictionary<string, object> saveData = new Dictionary<string, object>
        {
            ["player_data"] = playerData,
            ["level"] = level,
            ["score"] = score,
            ["timestamp"] = "2024-01-15 14:30:00"
        };
        this.saveData[saveKey] = saveData;
        Console.WriteLine($"Jeu sauvegardé dans le slot {currentSlot}");
    }
    
    public object LoadGame(int slot)
    {
        string saveKey = $"slot_{slot}";
        if (saveData.ContainsKey(saveKey))
        {
            Console.WriteLine($"Chargement du slot {slot}");
            return saveData[saveKey];
        }
        return null;
    }
    
    public void SetSaveSlot(int slot)
    {
        currentSlot = slot;
    }
}

// Utilisation
GameSaveManager saveManager1 = GameSaveManager.Instance;
GameSaveManager saveManager2 = GameSaveManager.Instance;
Console.WriteLine(ReferenceEquals(saveManager1, saveManager2)); // True - même instance

saveManager1.Initialize();
saveManager1.SaveGame(new Dictionary<string, object> { ["name"] = "Joueur1", ["health"] = 100 }, 5, 1500);
saveManager2.LoadGame(1); // Utilise la même instance
```

### 1.2 Notification d'evenements

**Problème :** Vous voulez notifier plusieurs systèmes du jeu quand un événement se produit (joueur meurt, niveau terminé, objet collecté), sans créer un couplage fort entre l'objet émetteur et les récepteurs.

**Solution :** Utilisez le patron **Observateur** pour établir une relation un-à-plusieurs entre objets.

**Exemple de code :**

```csharp
public interface IGameObserver
{
    void OnEvent(string eventType, Dictionary<string, object> data);
}

public class GameEventSystem
{
    private List<IGameObserver> _observers = new List<IGameObserver>();
    
    public void Subscribe(IGameObserver observer)
    {
        _observers.Add(observer);
    }
    
    public void Unsubscribe(IGameObserver observer)
    {
        _observers.Remove(observer);
    }
    
    public void Notify(string eventType, Dictionary<string, object> data)
    {
        foreach (var observer in _observers)
        {
            observer.OnEvent(eventType, data);
        }
    }
}

public class AudioSystem : IGameObserver
{
    public void OnEvent(string eventType, Dictionary<string, object> data)
    {
        switch (eventType)
        {
            case "player_death":
                Console.WriteLine("🔊 Son de mort joué");
                break;
            case "level_complete":
                Console.WriteLine("🔊 Musique de victoire jouée");
                break;
            case "item_collected":
                Console.WriteLine("🔊 Son de collecte joué");
                break;
        }
    }
}

public class UISystem : IGameObserver
{
    public void OnEvent(string eventType, Dictionary<string, object> data)
    {
        switch (eventType)
        {
            case "player_death":
                Console.WriteLine("📱 Affichage de l'écran Game Over");
                break;
            case "level_complete":
                Console.WriteLine("📱 Affichage du score final");
                break;
            case "item_collected":
                Console.WriteLine($"📱 Mise à jour de l'inventaire: {data["item"]}");
                break;
        }
    }
}

public class AchievementSystem : IGameObserver
{
    public void OnEvent(string eventType, Dictionary<string, object> data)
    {
        switch (eventType)
        {
            case "item_collected":
                Console.WriteLine("🏆 Succès débloqué: Collectionneur!");
                break;
            case "level_complete":
                Console.WriteLine($"🏆 Succès débloqué: Niveau {data["level"]} terminé!");
                break;
        }
    }
}

// Utilisation
var eventSystem = new GameEventSystem();
var audio = new AudioSystem();
var ui = new UISystem();
var achievements = new AchievementSystem();

eventSystem.Subscribe(audio);
eventSystem.Subscribe(ui);
eventSystem.Subscribe(achievements);

// Simulation d'événements de jeu
eventSystem.Notify("item_collected", new Dictionary<string, object> { ["item"] = "Épée légendaire" });
eventSystem.Notify("level_complete", new Dictionary<string, object> { ["level"] = 3, ["score"] = 2500 });
eventSystem.Notify("player_death", new Dictionary<string, object> { ["cause"] = "chute" });
```

### 1.3 Creation d'objets complexes

**Problème :** Vous devez créer des ennemis, armes ou objets dans un jeu vidéo dont le processus de création est complexe ou dépend de paramètres dynamiques (niveau du joueur, zone du jeu, difficulté).

**Solution :** Utilisez le patron **Factory** pour encapsuler la logique de création d'objets.

**Exemple de code :**

```csharp
public abstract class Enemy
{
    public abstract string Attack();
    public abstract Dictionary<string, object> GetStats();
}

public class Goblin : Enemy
{
    public int Level { get; private set; }
    public int Health { get; private set; }
    public int Damage { get; private set; }
    
    public Goblin(int level)
    {
        Level = level;
        Health = 50 + (level * 10);
        Damage = 15 + (level * 5);
    }
    
    public override string Attack()
    {
        return $"Goblin niveau {Level} attaque pour {Damage} dégâts!";
    }
    
    public override Dictionary<string, object> GetStats()
    {
        return new Dictionary<string, object>
        {
            ["health"] = Health,
            ["damage"] = Damage,
            ["type"] = "Goblin"
        };
    }
}

public class Orc : Enemy
{
    public int Level { get; private set; }
    public int Health { get; private set; }
    public int Damage { get; private set; }
    
    public Orc(int level)
    {
        Level = level;
        Health = 100 + (level * 15);
        Damage = 25 + (level * 8);
    }
    
    public override string Attack()
    {
        return $"Orc niveau {Level} attaque pour {Damage} dégâts!";
    }
    
    public override Dictionary<string, object> GetStats()
    {
        return new Dictionary<string, object>
        {
            ["health"] = Health,
            ["damage"] = Damage,
            ["type"] = "Orc"
        };
    }
}

public class Dragon : Enemy
{
    public int Level { get; private set; }
    public int Health { get; private set; }
    public int Damage { get; private set; }
    
    public Dragon(int level)
    {
        Level = level;
        Health = 500 + (level * 50);
        Damage = 100 + (level * 20);
    }
    
    public override string Attack()
    {
        return $"Dragon niveau {Level} crache du feu pour {Damage} dégâts!";
    }
    
    public override Dictionary<string, object> GetStats()
    {
        return new Dictionary<string, object>
        {
            ["health"] = Health,
            ["damage"] = Damage,
            ["type"] = "Dragon"
        };
    }
}

public class EnemyFactory
{
    public static Enemy CreateEnemy(string enemyType, int playerLevel, int zoneDifficulty = 1)
    {
        // Calcul du niveau de l'ennemi basé sur le niveau du joueur et la difficulté de la zone
        int enemyLevel = Math.Max(1, playerLevel + zoneDifficulty - 1);
        
        switch (enemyType.ToLower())
        {
            case "goblin":
                return new Goblin(enemyLevel);
            case "orc":
                return new Orc(enemyLevel);
            case "dragon":
                return new Dragon(enemyLevel);
            default:
                throw new ArgumentException($"Type d'ennemi non supporté: {enemyType}");
        }
    }
}

// Utilisation
EnemyFactory factory = new EnemyFactory();
int playerLevel = 5;

// Création d'ennemis adaptés au niveau du joueur
Enemy goblin = EnemyFactory.CreateEnemy("goblin", playerLevel, 1);  // Zone facile
Enemy orc = EnemyFactory.CreateEnemy("orc", playerLevel, 2);        // Zone moyenne
Enemy dragon = EnemyFactory.CreateEnemy("dragon", playerLevel, 3);  // Zone difficile

Console.WriteLine(goblin.Attack());
Console.WriteLine(orc.Attack());
Console.WriteLine(dragon.Attack());
Console.WriteLine($"Stats du gobelin: {string.Join(", ", goblin.GetStats().Select(kv => $"{kv.Key}: {kv.Value}"))}");
```

### 1.4 Creation de familles d'objets

**Problème :** Vous devez créer des familles d'objets liés dans un jeu vidéo qui doivent être compatibles entre eux (ex: armes et armures pour différentes factions, véhicules et armes pour différentes époques).

**Solution :** Utilisez le patron **Abstract Factory** pour créer des familles d'objets sans spécifier leurs classes concrètes.

**Exemple de code :**

```csharp
public abstract class Weapon
{
    public abstract string Attack();
}

public abstract class Armor
{
    public abstract string Defend();
}

// Armes et armures médiévales
public class MedievalSword : Weapon
{
    public override string Attack()
    {
        return "⚔️ Coup d'épée médiévale - 25 dégâts";
    }
}

public class MedievalShield : Armor
{
    public override string Defend()
    {
        return "🛡️ Protection du bouclier médiéval - 15 défense";
    }
}

// Armes et armures futuristes
public class LaserRifle : Weapon
{
    public override string Attack()
    {
        return "🔫 Tir de laser - 40 dégâts";
    }
}

public class EnergyShield : Armor
{
    public override string Defend()
    {
        return "⚡ Bouclier d'énergie - 25 défense";
    }
}

// Armes et armures modernes
public class AssaultRifle : Weapon
{
    public override string Attack()
    {
        return "🔫 Rafale d'assaut - 30 dégâts";
    }
}

public class BulletproofVest : Armor
{
    public override string Defend()
    {
        return "🦺 Protection du gilet pare-balles - 20 défense";
    }
}

public abstract class EquipmentFactory
{
    public abstract Weapon CreateWeapon();
    public abstract Armor CreateArmor();
}

public class MedievalFactory : EquipmentFactory
{
    public override Weapon CreateWeapon()
    {
        return new MedievalSword();
    }
    
    public override Armor CreateArmor()
    {
        return new MedievalShield();
    }
}

public class FuturisticFactory : EquipmentFactory
{
    public override Weapon CreateWeapon()
    {
        return new LaserRifle();
    }
    
    public override Armor CreateArmor()
    {
        return new EnergyShield();
    }
}

public class ModernFactory : EquipmentFactory
{
    public override Weapon CreateWeapon()
    {
        return new AssaultRifle();
    }
    
    public override Armor CreateArmor()
    {
        return new BulletproofVest();
    }
}

// Utilisation
public static (Weapon, Armor) EquipCharacter(EquipmentFactory factory, string characterName)
{
    Weapon weapon = factory.CreateWeapon();
    Armor armor = factory.CreateArmor();
	
    Console.WriteLine($"🎮 {characterName} équipé:");
    Console.WriteLine($"   Arme: {weapon.Attack()}");
    Console.WriteLine($"   Armure: {armor.Defend()}");
	
    return (weapon, armor);
}

// Création d'équipements pour différentes époques
MedievalFactory medievalFactory = new MedievalFactory();
FuturisticFactory futuristicFactory = new ();
ModernFactory modernFactory = new ModernFactory();

EquipCharacter(medievalFactory, "Chevalier");
EquipCharacter(futuristicFactory, "Soldat de l'espace");
EquipCharacter(modernFactory, "Soldat moderne");
```

### 1.5 Integration d'interfaces incompatibles

**Problème :** Vous avez des systèmes de jeu existants avec des interfaces qui ne correspondent pas à ce dont vous avez besoin (ex: ancien système de sauvegarde, API de tiers pour les graphiques).

**Solution :** Utilisez le patron **Adaptateur** pour faire fonctionner ensemble des classes avec des interfaces incompatibles.

**Exemple de code :**

```csharp
// Ancien système de rendu (legacy)
public class LegacyRenderer
{
    public string DrawOldStyle(int x, int y, string spriteId)
    {
        return $"Legacy: Dessin du sprite {spriteId} à ({x}, {y})";
    }
}

// Nouveau système de rendu
public class ModernRenderer
{
    public string RenderSprite((int x, int y) position, Dictionary<string, object> spriteData)
    {
        return $"Modern: Rendu du sprite {spriteData["name"]} à {position}";
    }
}

// Système de rendu unifié (interface cible)
public abstract class GameRenderer
{
    public abstract string Render(object entity);
}

// Adaptateur pour l'ancien système
public class LegacyRendererAdapter : GameRenderer
{
    private readonly LegacyRenderer _legacyRenderer;
    
    public LegacyRendererAdapter(LegacyRenderer legacyRenderer)
    {
        _legacyRenderer = legacyRenderer;
    }
    
    public override string Render(object entity)
    {
        // Adaptation de l'interface legacy vers l'interface moderne
        if (entity is LegacyEntity legacyEntity)
        {
            return _legacyRenderer.DrawOldStyle(legacyEntity.X, legacyEntity.Y, legacyEntity.SpriteId);
        }
        return "Erreur: Format d'entité non supporté";
    }
}

// Adaptateur pour le nouveau système
public class ModernRendererAdapter : GameRenderer
{
    private readonly ModernRenderer _modernRenderer;
    
    public ModernRendererAdapter(ModernRenderer modernRenderer)
    {
        _modernRenderer = modernRenderer;
    }
    
    public override string Render(object entity)
    {
        // Adaptation de l'interface moderne vers l'interface unifiée
        if (entity is ModernEntity modernEntity)
        {
            return _modernRenderer.RenderSprite(modernEntity.Position, modernEntity.SpriteData);
        }
        return "Erreur: Format d'entité non supporté";
    }
}

// Entités avec différentes interfaces
public class LegacyEntity
{
    public int X { get; set; }
    public int Y { get; set; }
    public string SpriteId { get; set; }
    
    public LegacyEntity(int x, int y, string spriteId)
    {
        X = x;
        Y = y;
        SpriteId = spriteId;
    }
}

public class ModernEntity
{
    public (int x, int y) Position { get; set; }
    public Dictionary<string, object> SpriteData { get; set; }
    
    public ModernEntity((int x, int y) position, Dictionary<string, object> spriteData)
    {
        Position = position;
        SpriteData = spriteData;
    }
}

// Utilisation
LegacyRenderer legacyRenderer = new LegacyRenderer();
ModernRenderer modernRenderer = new ModernRenderer();

// Création des adaptateurs
LegacyRendererAdapter legacyAdapter = new LegacyRendererAdapter(legacyRenderer);
ModernRendererAdapter modernAdapter = new ModernRendererAdapter(modernRenderer);

// Entités avec interfaces différentes
LegacyEntity legacyEntity = new LegacyEntity(100, 200, "player_sprite");
ModernEntity modernEntity = new ModernEntity((150, 250), new Dictionary<string, object> 
{ 
    ["name"] = "enemy_sprite", 
    ["texture"] = "enemy.png" 
});

// Rendu unifié grâce aux adaptateurs
var renderers = new[] { legacyAdapter, modernAdapter };
var entities = new object[] { legacyEntity, modernEntity };

for (int i = 0; i < renderers.Length; i++)
{
    Console.WriteLine(renderers[i].Render(entities[i]));
}
```

### 1.6 Extension de fonctionnalites

**Problème :** Vous voulez ajouter de nouvelles fonctionnalités à des armes ou objets dans un jeu vidéo sans modifier leur classe de base (enchantements, améliorations, effets spéciaux).

**Solution :** Utilisez le patron **Décorateur** pour envelopper un objet et lui ajouter des comportements dynamiquement.

**Exemple de code :**

```csharp
public abstract class Weapon
{
    public abstract int GetDamage();
    public abstract string GetDescription();
}

public class BasicSword : Weapon
{
    public override int GetDamage()
    {
        return 20;
    }
    
    public override string GetDescription()
    {
        return "Épée basique";
    }
}

public abstract class WeaponDecorator : Weapon
{
    protected Weapon _weapon;
    
    public WeaponDecorator(Weapon weapon)
    {
        _weapon = weapon;
    }
    
    public override int GetDamage()
    {
        return _weapon.GetDamage();
    }
    
    public override string GetDescription()
    {
        return _weapon.GetDescription();
    }
}

public class FireEnchantment : WeaponDecorator
{
    public FireEnchantment(Weapon weapon) : base(weapon) { }
    
    public override int GetDamage()
    {
        return _weapon.GetDamage() + 10;
    }
    
    public override string GetDescription()
    {
        return _weapon.GetDescription() + " + Enchantement de feu";
    }
}

public class SharpnessEnchantment : WeaponDecorator
{
    public SharpnessEnchantment(Weapon weapon) : base(weapon) { }
    
    public override int GetDamage()
    {
        return _weapon.GetDamage() + 15;
    }
    
    public override string GetDescription()
    {
        return _weapon.GetDescription() + " + Tranchant";
    }
}

public class PoisonEnchantment : WeaponDecorator
{
    public PoisonEnchantment(Weapon weapon) : base(weapon) { }
    
    public override int GetDamage()
    {
        return _weapon.GetDamage() + 5;
    }
    
    public override string GetDescription()
    {
        return _weapon.GetDescription() + " + Poison";
    }
}

public class DurabilityUpgrade : WeaponDecorator
{
    public DurabilityUpgrade(Weapon weapon) : base(weapon) { }
    
    public override int GetDamage()
    {
        return _weapon.GetDamage() + 8;
    }
    
    public override string GetDescription()
    {
        return _weapon.GetDescription() + " + Renforcée";
    }
}

// Utilisation
BasicSword sword = new BasicSword();
Console.WriteLine($"Arme de base: {sword.GetDescription()} - {sword.GetDamage()} dégâts");

// Ajout d'enchantements
sword = new FireEnchantment(sword);
Console.WriteLine($"Avec feu: {sword.GetDescription()} - {sword.GetDamage()} dégâts");

sword = new SharpnessEnchantment(sword);
Console.WriteLine($"Avec tranchant: {sword.GetDescription()} - {sword.GetDamage()} dégâts");

sword = new PoisonEnchantment(sword);
Console.WriteLine($"Avec poison: {sword.GetDescription()} - {sword.GetDamage()} dégâts");

// Création d'une épée différente avec d'autres améliorations
BasicSword legendarySword = new BasicSword();
legendarySword = new DurabilityUpgrade(legendarySword);
legendarySword = new SharpnessEnchantment(legendarySword);
Console.WriteLine($"Épée légendaire: {legendarySword.GetDescription()} - {legendarySword.GetDamage()} dégâts");
```

### 1.7 Simplification d'interfaces complexes

**Problème :** Vous avez un système de jeu complexe avec de nombreuses classes et méthodes (graphiques, audio, physique, IA), et vous voulez fournir une interface simple pour les clients.

**Solution :** Utilisez le patron **Façade** pour fournir une interface unifiée à un sous-système complexe.

**Exemple de code :**

```csharp
public class GraphicsEngine
{
    public string InitializeRenderer()
    {
        return "🎨 Moteur graphique initialisé";
    }
    
    public string LoadTextures(string[] textureList)
    {
        return $"🖼️ Chargement de {textureList.Length} textures";
    }
    
    public string RenderFrame(string[] sceneData)
    {
        return $"📺 Rendu de la frame avec {sceneData.Length} objets";
    }
}

public class AudioEngine
{
    public string InitializeAudio()
    {
        return "🔊 Moteur audio initialisé";
    }
    
    public string LoadSounds(string[] soundList)
    {
        return $"🎵 Chargement de {soundList.Length} sons";
    }
    
    public string PlayBackgroundMusic(string track)
    {
        return $"🎶 Lecture de la musique: {track}";
    }
}

public class PhysicsEngine
{
    public string InitializePhysics()
    {
        return "⚡ Moteur physique initialisé";
    }
    
    public string CreateWorld(double gravity)
    {
        return $"🌍 Monde physique créé avec gravité: {gravity}";
    }
    
    public string SimulateStep(double deltaTime)
    {
        return $"🔄 Simulation physique (Δt={deltaTime}s)";
    }
}

public class AIEngine
{
    public string InitializeAI()
    {
        return "🧠 Moteur IA initialisé";
    }
    
    public string LoadBehaviorTrees(string[] trees)
    {
        return $"🌳 Chargement de {trees.Length} arbres de comportement";
    }
    
    public string UpdateAI(string[] entities)
    {
        return $"🤖 Mise à jour IA pour {entities.Length} entités";
    }
}

public class GameEngineFacade
{
    private readonly GraphicsEngine _graphics;
    private readonly AudioEngine _audio;
    private readonly PhysicsEngine _physics;
    private readonly AIEngine _ai;
    
    public GameEngineFacade()
    {
        _graphics = new GraphicsEngine();
        _audio = new AudioEngine();
        _physics = new PhysicsEngine();
        _ai = new AIEngine();
    }
    
    public string StartGame(Dictionary<string, object> gameConfig)
    {
        // Interface simplifiée pour démarrer le jeu
        Console.WriteLine("🚀 Démarrage du jeu...");
        
        // Initialisation des sous-systèmes
        Console.WriteLine(_graphics.InitializeRenderer());
        Console.WriteLine(_audio.InitializeAudio());
        Console.WriteLine(_physics.InitializePhysics());
        Console.WriteLine(_ai.InitializeAI());
        
        // Configuration des ressources
        Console.WriteLine(_graphics.LoadTextures((string[])gameConfig["textures"]));
        Console.WriteLine(_audio.LoadSounds((string[])gameConfig["sounds"]));
        Console.WriteLine(_physics.CreateWorld((double)gameConfig["gravity"]));
        Console.WriteLine(_ai.LoadBehaviorTrees((string[])gameConfig["ai_trees"]));
        
        return "✅ Jeu prêt à jouer!";
    }
    
    public string UpdateGame(double deltaTime, string[] sceneData, string[] entities)
    {
        // Interface simplifiée pour la boucle de jeu
        Console.WriteLine(_graphics.RenderFrame(sceneData));
        Console.WriteLine(_physics.SimulateStep(deltaTime));
        Console.WriteLine(_ai.UpdateAI(entities));
        return "🎮 Frame mise à jour";
    }
}

// Utilisation
GameEngineFacade gameEngine = new GameEngineFacade();

// Configuration du jeu
Dictionary<string, object> config = new Dictionary<string, object>
{
    ["textures"] = new[] { "player.png", "enemy.png", "background.jpg" },
    ["sounds"] = new[] { "jump.wav", "collect.wav", "music.mp3" },
    ["gravity"] = -9.81,
    ["ai_trees"] = new[] { "enemy_behavior", "npc_behavior" }
};

// Démarrage simplifié
string result = gameEngine.StartGame(config);
Console.WriteLine(result);

// Boucle de jeu simplifiée
string[] scene = { "joueur", "ennemi1", "ennemi2", "objet" };
string[] entities = { "ennemi1", "ennemi2", "npc1" };
gameEngine.UpdateGame(0.016, scene, entities);
```

### 1.8 Traitement d'objets composites

**Problème :** Vous voulez traiter des objets individuels et des compositions d'objets de manière uniforme dans un jeu vidéo (inventaire, quêtes, niveaux).

**Solution :** Utilisez le patron **Composite** pour composer des objets en structures arborescentes.

**Exemple de code :**

```csharp
public abstract class GameItem
{
    public abstract int GetValue();
    public abstract string GetDescription();
    public abstract void Display(int indent = 0);
}

public class Item : GameItem
{
    public string Name { get; private set; }
    public int Value { get; private set; }
    public string Description { get; private set; }
    
    public Item(string name, int value, string description)
    {
        Name = name;
        Value = value;
        Description = description;
    }
    
    public override int GetValue()
    {
        return Value;
    }
    
    public override string GetDescription()
    {
        return Description;
    }
    
    public override void Display(int indent = 0)
    {
        Console.WriteLine(new string(' ', indent * 2) + $"💎 {Name} - {Value} or - {Description}");
    }
}

public class Container : GameItem
{
    public string Name { get; private set; }
    public string Description { get; private set; }
    private List<GameItem> _items = new List<GameItem>();
    
    public Container(string name, string description)
    {
        Name = name;
        Description = description;
    }
    
    public void AddItem(GameItem item)
    {
        _items.Add(item);
    }
    
    public override int GetValue()
    {
        return _items.Sum(item => item.GetValue());
    }
    
    public override string GetDescription()
    {
        return $"{Description} (contient {_items.Count} objets)";
    }
    
    public override void Display(int indent = 0)
    {
        Console.WriteLine(new string(' ', indent * 2) + $"🎒 {Name} - {GetValue()} or total");
        foreach (var item in _items)
        {
            item.Display(indent + 1);
        }
    }
}

// Utilisation
// Création d'objets individuels
Item sword = new Item("Épée légendaire", 100, "Une épée magique puissante");
Item potion = new Item("Potion de soin", 25, "Restaure 50 points de vie");
Item gem = new Item("Gemme de feu", 50, "Pierre précieuse rougeoyante");

// Création de conteneurs
Container treasureChest = new Container("Coffre au trésor", "Un coffre mystérieux");
Container playerInventory = new Container("Inventaire du joueur", "Sac à dos du héros");
Container magicBag = new Container("Sac magique", "Sac enchanté pour objets précieux");

// Ajout d'objets dans les conteneurs
treasureChest.AddItem(sword);
treasureChest.AddItem(gem);

magicBag.AddItem(potion);
magicBag.AddItem(new Item("Parchemin de téléportation", 75, "Permet de se téléporter"));

playerInventory.AddItem(treasureChest);
playerInventory.AddItem(magicBag);
playerInventory.AddItem(new Item("Clé dorée", 10, "Ouvre des portes spéciales"));

// Affichage de la structure
Console.WriteLine("🎮 Inventaire du joueur:");
playerInventory.Display();
Console.WriteLine($"\n💰 Valeur totale de l'inventaire: {playerInventory.GetValue()} or");
```

### 1.9 Choix d'algorithmes

**Problème :** Vous avez plusieurs algorithmes pour accomplir une tâche dans un jeu vidéo et vous voulez pouvoir les interchanger dynamiquement (IA d'ennemis, systèmes de pathfinding, calculs de dégâts).

**Solution :** Utilisez le patron **Strategy** pour définir une famille d'algorithmes et les rendre interchangeables.

**Exemple de code :**

```csharp
public interface IAIStrategy
{
    string CalculateMove(Enemy enemy, Player player);
}

public class AggressiveAI : IAIStrategy
{
    public string CalculateMove(Enemy enemy, Player player)
    {
        // IA agressive : attaque directement
        int distance = Math.Abs(enemy.Position - player.Position);
        if (distance <= enemy.AttackRange)
        {
            return $"🤺 {enemy.Name} attaque le joueur!";
        }
        else
        {
            return $"🏃 {enemy.Name} court vers le joueur";
        }
    }
}

public class DefensiveAI : IAIStrategy
{
    public string CalculateMove(Enemy enemy, Player player)
    {
        // IA défensive : garde sa position et contre-attaque
        int distance = Math.Abs(enemy.Position - player.Position);
        if (distance <= enemy.AttackRange)
        {
            return $"🛡️ {enemy.Name} se défend et contre-attaque!";
        }
        else
        {
            return $"⏸️ {enemy.Name} reste en position défensive";
        }
    }
}

public class StealthAI : IAIStrategy
{
    public string CalculateMove(Enemy enemy, Player player)
    {
        // IA furtive : se cache et attaque par surprise
        int distance = Math.Abs(enemy.Position - player.Position);
        if (distance <= enemy.AttackRange)
        {
            return $"🗡️ {enemy.Name} attaque par surprise!";
        }
        else
        {
            return $"👻 {enemy.Name} se cache et se rapproche furtivement";
        }
    }
}

public class SwarmAI : IAIStrategy
{
    public string CalculateMove(Enemy enemy, Player player)
    {
        // IA d'essaim : coordonne avec d'autres ennemis
        return $"🐝 {enemy.Name} coordonne l'attaque en groupe!";
    }
}

public class Enemy
{
    public string Name { get; private set; }
    public int Position { get; private set; }
    public int AttackRange { get; private set; }
    private IAIStrategy _aiStrategy;
    
    public Enemy(string name, int position, int attackRange)
    {
        Name = name;
        Position = position;
        AttackRange = attackRange;
    }
    
    public void SetAIStrategy(IAIStrategy strategy)
    {
        _aiStrategy = strategy;
    }
    
    public string Act(Player player)
    {
        if (_aiStrategy != null)
        {
            return _aiStrategy.CalculateMove(this, player);
        }
        return $"{Name} ne sait pas quoi faire";
    }
}

public class Player
{
    public string Name { get; private set; }
    public int Position { get; private set; }
    
    public Player(string name, int position)
    {
        Name = name;
        Position = position;
    }
}

// Utilisation
Player player = new Player("Héros", 10);

// Création d'ennemis avec différentes stratégies
Enemy goblin = new Enemy("Goblin", 5, 2);
Enemy orc = new Enemy("Orc", 8, 3);
Enemy assassin = new Enemy("Assassin", 12, 1);
Enemy bees = new Enemy("Abeilles", 6, 1);

// Attribution de stratégies
goblin.SetAIStrategy(new AggressiveAI());
orc.SetAIStrategy(new DefensiveAI());
assassin.SetAIStrategy(new StealthAI());
bees.SetAIStrategy(new SwarmAI());

// Simulation de combat
Enemy[] enemies = new[] { goblin, orc, assassin, bees };

Console.WriteLine("⚔️ Combat en cours:");
foreach (Enemy enemy in enemies)
{
    Console.WriteLine(enemy.Act(player));
}

// Changement de stratégie dynamique
Console.WriteLine("\n🔄 Le gobelin change de stratégie...");
goblin.SetAIStrategy(new DefensiveAI());
Console.WriteLine(goblin.Act(player));
```

### 1.10 Gestion d'etats

**Problème :** Un objet change de comportement selon son état interne dans un jeu vidéo, et vous voulez éviter les conditions complexes (joueur blessé/en bonne santé, ennemi alerte/repos, objet activé/désactivé).

**Solution :** Utilisez le patron **State** pour permettre à un objet de modifier son comportement quand son état interne change.

**Exemple de code :**

```csharp
public interface IPlayerState
{
    string HandleAction(Player player, string action);
    string GetDescription();
}

public class HealthyState : IPlayerState
{
    public string HandleAction(Player player, string action)
    {
        switch (action)
        {
            case "take_damage":
                player.Health -= 20;
                if (player.Health <= 0)
                {
                    player.SetState(new DeadState());
                }
                else if (player.Health <= 30)
                {
                    player.SetState(new InjuredState());
                }
                return $"💚 {player.Name} prend 20 dégâts (PV: {player.Health})";
            
            case "heal":
                player.Health = Math.Min(100, player.Health + 30);
                return $"💚 {player.Name} se soigne (PV: {player.Health})";
            
            case "attack":
                return $"⚔️ {player.Name} attaque avec pleine puissance!";
            
            default:
                return $"💚 {player.Name} est en bonne santé";
        }
    }
    
    public string GetDescription()
    {
        return "En bonne santé";
    }
}

public class InjuredState : IPlayerState
{
    public string HandleAction(Player player, string action)
    {
        switch (action)
        {
            case "take_damage":
                player.Health -= 15;
                if (player.Health <= 0)
                {
                    player.SetState(new DeadState());
                }
                return $"🩹 {player.Name} prend 15 dégâts (PV: {player.Health})";
            
            case "heal":
                player.Health = Math.Min(100, player.Health + 20);
                if (player.Health > 30)
                {
                    player.SetState(new HealthyState());
                }
                return $"🩹 {player.Name} se soigne (PV: {player.Health})";
            
            case "attack":
                return $"⚔️ {player.Name} attaque avec difficulté (-50% dégâts)";
            
            default:
                return $"🩹 {player.Name} est blessé";
        }
    }
    
    public string GetDescription()
    {
        return "Blessé";
    }
}

public class DeadState : IPlayerState
{
    public string HandleAction(Player player, string action)
    {
        switch (action)
        {
            case "revive":
                player.Health = 50;
                player.SetState(new InjuredState());
                return $"💀 {player.Name} est ressuscité!";
            
            case "attack":
                return $"💀 {player.Name} ne peut pas attaquer (mort)";
            
            default:
                return $"💀 {player.Name} est mort";
        }
    }
    
    public string GetDescription()
    {
        return "Mort";
    }
}

public class Player
{
    public string Name { get; private set; }
    public int Health { get; set; }
    private IPlayerState _state;
    
    public Player(string name)
    {
        Name = name;
        Health = 100;
        _state = new HealthyState();
    }
    
    public void SetState(IPlayerState state)
    {
        _state = state;
    }
    
    public string Act(string action)
    {
        return _state.HandleAction(this, action);
    }
    
    public string GetStatus()
    {
        return $"{Name}: {_state.GetDescription()} (PV: {Health})";
    }
}

// Utilisation
var player = new Player("Héros");

Console.WriteLine("🎮 Simulation de combat:");
Console.WriteLine(player.GetStatus());

// Série d'actions
string[] actions = { "attack", "take_damage", "attack", "take_damage", "take_damage", "heal", "attack", "take_damage", "take_damage", "revive" };

foreach (string action in actions)
{
    string result = player.Act(action);
    Console.WriteLine($"Action: {action} -> {result}");
    Console.WriteLine($"État: {player.GetStatus()}");
    Console.WriteLine();
}
```

---

## Partie 2 : Similitudes, différences et risques de confusion

### 2.1 Strategy vs State

**Similitudes :**
- Les deux patrons utilisent la composition pour déléguer le comportement
- Ils permettent de changer le comportement d'un objet dynamiquement
- Ils évitent les conditions complexes (if/switch)

**Différences :**
- **Strategy** : Le client choisit l'algorithme à utiliser. L'objet context ne change pas de stratégie automatiquement.
- **State** : L'objet change d'état automatiquement selon ses transitions internes. Le client n'a pas besoin de connaître les états.

**Risques de confusion :**
- Confondre le moment où le changement se produit (manuel vs automatique)
- Utiliser State quand on a besoin de Strategy (et vice versa)

**Quand utiliser lequel :**
- **Strategy** : Quand vous voulez que le client choisisse l'algorithme
- **State** : Quand l'objet doit changer de comportement selon son état interne

### 2.2 Decorateur vs Façade vs Adaptateur

**Similitudes :**
- Tous trois sont des patrons structurels
- Ils modifient l'interface d'un objet existant
- Ils permettent de travailler avec des objets sans les modifier directement

**Différences :**
- **Décorateur** : Ajoute des fonctionnalités à un objet existant, enveloppe l'objet original
- **Façade** : Simplifie l'interface d'un sous-système complexe, ne modifie pas les objets existants
- **Adaptateur** : Fait fonctionner ensemble des interfaces incompatibles, traduit les appels

**Risques de confusion :**
- Utiliser Façade au lieu de Décorateur pour ajouter des fonctionnalités
- Confondre Adaptateur et Façade (l'Adaptateur traduit, la Façade simplifie)
- Sur-utiliser le Décorateur au lieu de l'héritage simple

**Quand utiliser lequel :**
- **Décorateur** : Pour ajouter des fonctionnalités dynamiquement
- **Façade** : Pour simplifier l'utilisation d'un système complexe
- **Adaptateur** : Pour faire fonctionner des interfaces incompatibles

### 2.3 Composite vs Decorateur

**Similitudes :**
- Les deux utilisent la composition et la récursion
- Ils permettent de traiter des objets de manière uniforme
- Ils peuvent être imbriqués

**Différences :**
- **Composite** : Structure hiérarchique d'objets (arbre), traite les objets individuels et les groupes de la même façon
- **Décorateur** : Enveloppe un objet pour ajouter des fonctionnalités, un seul objet à la fois

**Risques de confusion :**
- Utiliser Composite pour ajouter des fonctionnalités (au lieu de Décorateur)
- Utiliser Décorateur pour créer des structures hiérarchiques (au lieu de Composite)
- Confondre la structure (arbre vs chaîne)

**Quand utiliser lequel :**
- **Composite** : Pour représenter des structures hiérarchiques (fichiers/dossiers, menus)
- **Décorateur** : Pour ajouter des fonctionnalités à un objet existant

### 2.4 Factory vs Singleton

**Similitudes :**
- Les deux sont des patrons de création
- Ils contrôlent la création d'objets
- Ils peuvent être implémentés comme des méthodes statiques

**Différences :**
- **Factory** : Crée différents types d'objets selon des paramètres, peut créer plusieurs instances
- **Singleton** : Garantit qu'une seule instance d'une classe existe, pas de paramètres de création

**Risques de confusion :**
- Utiliser Singleton quand on a besoin de plusieurs instances
- Utiliser Factory pour garantir une seule instance (au lieu de Singleton)
- Mélanger les responsabilités (création vs unicité)

**Quand utiliser lequel :**
- **Factory** : Pour créer des objets selon des critères, avec possibilité de plusieurs instances
- **Singleton** : Pour garantir une seule instance d'une ressource partagée

---

## Conclusion

Ce guide présente les patrons de conception sous l'angle des problèmes qu'ils résolvent, permettant aux développeurs de choisir rapidement la solution appropriée. La compréhension des similitudes et différences entre patrons évite les confusions courantes et améliore la qualité du code.

Chaque patron a sa place dans l'arsenal du développeur, et le choix dépend du contexte spécifique du problème à résoudre.
