# Lab 23 — Anti-Debug Natif Android avec JNI et NDK

**Contexte pédagogique :** Ce laboratoire prolonge le TP JNI de base en ajoutant une couche défensive native dans une bibliothèque C++. L'objectif est de montrer comment déplacer des contrôles de sécurité dans le code natif Android, où ils sont plus difficiles à analyser que dans du bytecode Java classique. Le lab reste pédagogique et orienté observation — pas une protection absolue, mais une démonstration crédible de durcissement mobile.

---

## Environnement

| Composant | Détail |
|---|---|
| IDE | Android Studio |
| Langage Java | Java |
| Langage natif | C++ (NDK) |
| Système de build natif | CMake |
| Bibliothèque système | `liblog` (Logcat natif) |
| Projet de base | JNIDemo (lab JNI précédent) |
| ABI cible | arm64-v8a / x86_64 (émulateur) |

---

## Architecture du projet

```
JNIDemo/
├── app/src/main/
│   ├── cpp/
│   │   ├── native-lib.cpp       ← logique native + contrôles défensifs
│   │   └── CMakeLists.txt       ← configuration du build natif
│   ├── java/.../MainActivity.java
│   └── res/layout/activity_main.xml
```

### Flux d'exécution

```
MainActivity (Java)
       |
       | appelle isDebugDetected()
       ↓
libnative-lib.so (C++)
       |
       ├── isBeingTraced()           → contrôle ptrace
       └── containsSuspiciousLibs() → inspection /proc/self/maps
       |
       ↓ retourne jboolean
MainActivity adapte l'interface (OK ou ALERTE)
```

---

## Partie 1 — Configuration CMake

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("jnidemo")

add_library(native-lib SHARED native-lib.cpp)

find_library(log-lib log)

target_link_libraries(native-lib ${log-lib})
```

`find_library(log-lib log)` récupère la bibliothèque Android `log` nécessaire pour écrire dans Logcat depuis le code C++.

---

## Partie 2 — Code natif (native-lib.cpp)

### Macros de log natif

```cpp
#define LOG_TAG "ANTI_DEBUG"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,  LOG_TAG, __VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,  LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
```

Permettent de filtrer les messages dans Logcat avec le tag `ANTI_DEBUG`.

---

### Contrôle 1 — Détection via ptrace

```cpp
static bool isBeingTraced() {
    long result = ptrace(PTRACE_TRACEME, 0, 0, 0);
    if (result == -1) {
        LOGE("Etat suspect : trace/debug detecte");
        return true;
    }
    LOGI("Aucun trace/debug detecte via ptrace");
    return false;
}
```

**Principe :** `PTRACE_TRACEME` demande au processus de se déclarer traçable. Si un débogueur est déjà attaché, l'appel échoue et retourne `-1`. C'est un signal bas niveau indiquant un contexte supervisé.

> ⚠️ Ce signal peut varier selon l'ABI, le mode de lancement et les politiques du système. Il s'agit d'un indicateur parmi d'autres, pas d'une certitude absolue.

---

### Contrôle 2 — Inspection de `/proc/self/maps`

```cpp
static bool containsSuspiciousLibraryNames() {
    FILE* maps = fopen("/proc/self/maps", "r");
    char line[512];
    while (fgets(line, sizeof(line), maps)) {
        if (strstr(line, "frida")   ||
            strstr(line, "xposed")  ||
            strstr(line, "gdbserver")||
            strstr(line, "magisk")) {
            LOGE("Signature suspecte trouvee : %s", line);
            fclose(maps);
            return true;
        }
    }
    fclose(maps);
    LOGI("Aucune signature suspecte dans /proc/self/maps");
    return false;
}
```

**Principe :** `/proc/self/maps` liste toutes les régions mémoire et bibliothèques chargées dans le processus. La présence de chaînes comme `frida`, `xposed` ou `magisk` indique un environnement d'instrumentation dynamique ou de root.

---

### Fonction JNI principale

```cpp
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_example_jnidemo_MainActivity_isDebugDetected(JNIEnv* env, jobject) {
    bool traced         = isBeingTraced();
    bool suspiciousMaps = containsSuspiciousLibraryNames();

    if (traced || suspiciousMaps) {
        LOGE("Etat de securite : DEBUG / INSTRUMENTATION detecte");
        return JNI_TRUE;
    }
    LOGI("Etat de securite : OK");
    return JNI_FALSE;
}
```

Fusionne les deux signaux et retourne un `jboolean` lisible côté Java. La décision finale (bloquer, afficher, fermer) reste du côté Java.

---

## Partie 3 — Côté Java (MainActivity.java)

```java
public native boolean isDebugDetected();

boolean suspicious = isDebugDetected();

if (suspicious) {
    tvStatus.setText("Etat securite : environnement suspect detecte");
    tvStatus.setTextColor(Color.RED);
    tvHello.setText("Fonction native sensible desactivee");
    tvFact.setText("Calcul natif bloque");
} else {
    tvStatus.setText("Etat securite : OK");
    tvStatus.setTextColor(Color.parseColor("#2E7D32"));
    tvHello.setText(helloFromJNI());
    tvFact.setText("Factoriel de 10 = " + factorial(10));
}
```

**Pourquoi cette approche :** plutôt qu'un crash brutal, l'application adapte son interface. C'est plus propre, plus observable et plus adapté à un contexte de TP.

---

## Observation Logcat

Filtrer avec le tag `ANTI_DEBUG` dans Logcat.

| Situation | Message attendu |
|---|---|
| Lancement normal | `Etat de securite : OK` |
| Débogueur attaché | `Etat suspect : trace/debug detecte` |
| Frida/Xposed présent | `Signature suspecte trouvee dans maps` |
| Combiné | `DEBUG / INSTRUMENTATION detecte` |

---

## Points clés retenus

- Le code natif C++ n'est pas dans le bytecode Java — il est plus difficile à analyser statiquement avec des outils comme jadx.
- `ptrace(PTRACE_TRACEME)` est un signal défensif bas niveau exploitant le fait qu'un processus ne peut être tracé que par un seul outil à la fois.
- `/proc/self/maps` est une source d'information précieuse : elle révèle la présence de bibliothèques d'instrumentation comme Frida chargées en mémoire.
- Retourner un `jboolean` depuis le natif et laisser Java décider de la réaction est un pattern propre et testable.
- Un seul contrôle est insuffisant — une défense robuste combine plusieurs signaux indépendants.
- Les logs natifs via `__android_log_print` sont indispensables pour observer le comportement du code C++ pendant le développement.

---

## Limites de cette approche

- **Faux positifs** : certains environnements de développement peuvent déclencher des alertes sans intention malveillante.
- **Détection imparfaite** : un attaquant expérimenté peut patcher les appels `ptrace` ou renommer les bibliothèques Frida pour contourner les checks.
- **Strings en clair** : les chaînes `"frida"`, `"xposed"` dans le binaire sont visibles via `strings` — en production elles seraient obfusquées.

---

*Lab réalisé dans le cadre du cours Sécurité des Applications Mobiles — ENSA Marrakech, Filière GCDSTE*
