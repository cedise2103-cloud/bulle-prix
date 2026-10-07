# bulle-prixnom : Créer un APK
sur:
  workflow_dispatch :
  pousser:
    branches : [principale]
autorisations :
  contenu : écrire
environnement :
  OPTIONS DE L'OUTIL JAVA : -Dfile.encoding=UTF-8
  LC_ALL : C.UTF-8
emplois :
  construire:
    fonctionne sous : ubuntu-latest
    mesures:
      - utilise : actions/setup-java@v4
        avec:
          distribution : témurine
          version Java : '17'
      - utilise : gradle/actions/setup-gradle@v4
        avec:
          version de Gradle : '8.7'
      - nom : Générer le projet
        exécuter : |
          mkdir -p VeepeeBubble
          chat > ​​VeepeeBubble/README.txt <<'FILE_EOF'
          Bulle Prix - prototype (lecture de l'écran via accessibilite)

          1. Installez Android Studio, puis Fichier > Ouvrir > dossier VeepeeBubble.
          2. Brancher le téléphone (débogage USB) ou créer l'APK : Build > Build APK(s).
          3. Installez l'APK. Android 13+ : Paramètres > Applications > Bulle Prix > menu "..." >
             "Autoriser les paramètres restreints", puis activer le service dans Accessibilité.
          4. Ouvrir Veepee sur une fiche produit : la bulle affichant le prix lu.
             Appui long sur la bulle = copie du texte lu (à me coller pour ajuster la détection).
          FICHIER_EOF
          mkdir -p VeepeeBubble/app
          cat > VeepeeBubble/app/build.gradle.kts <<'FILE_EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          Android {
              espace de noms = "fr.ced.bubble"
              compileSdk = 34
              defaultConfig {
                  applicationId = "fr.ced.bubble"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "0.1"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          FICHIER_EOF
          mkdir -p VeepeeBubble/app/src/main
          chat > ​​VeepeeBubble/app/src/main/AndroidManifest.xml <<'FILE_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <application
                  android:label="@string/app_name"
                  android:theme="@android:style/Theme.Material.Light">
                  <activity android:name=".MainActivity" android:exported="true">
                      <filtre d'intention>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activité>
                  <service
                      android:name=".PriceAccessibilityService"
                      android:exported="true"
                      android:label="@string/app_name"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <filtre d'intention>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <méta-données
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifeste>
          FICHIER_EOF
          mkdir -p VeepeeBubble/app/src/main/java/fr/ced/bubble
          chat > ​​VeepeeBubble/app/src/main/java/fr/ced/bubble/MainActivity.kt <<'FILE_EOF'
          paquet fr.ced.bubble

          importer android.app.Activity
          importer android.content.Intent
          importer android.os.Bundle
          importer android.provider.Settings
          importer android.widget.Button
          importer android.widget.EditText
          importer android.widget.LinearLayout
          importer android.widget.TextView
          importer android.widget.Toast

          classe MainActivity : Activity() {
              remplacer fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val prefs = getSharedPreferences("cfg", MODE_PRIVATE)
                  val pad = (20 * resources.displayMetrics.density).toInt()

                  val layout = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      définirPadding(pad, pad, pad, pad)
                  }
                  layout.addView(TextView(this).apply {
                      text = "Bulle Prix (prototype)\n\n1) Active le service d'accessibilité Â« Bulle Prix Â».\n2) Ouvre Veepee : une bulle affiche le prix lu à l'écran.\n3) Appui long sur la bulle = copier le texte lu."
                      taille du texte = 16f
                  })
                  layout.addView(Button(this).apply {
                      text = "Ouvrir les réglages d'accessibilité"
                      setOnClickListener { startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)) }
                  })
                  layout.addView(TextView(this).apply {
                      text = "\nFiltre d'appli (ex : veepee). Vide = toutes les applications."
                  })
                  val input = EditText(this).apply {
                      setText(prefs.getString("filter", ""))
                      indice = "veepee"
                      définirLigneUnique()
                  }
                  mise en page.ajouterView(entrée)
                  layout.addView(Button(this).apply {
                      text = "Enregistrer le filtre"
                      setOnClickListener {
                          prefs.edit().putString("filter", input.text.toString().trim()).apply()
                          Toast.makeText(context, "Enregistré", Toast.LENGTH_SHORT).show()
                      }
                  })
                  définirContentView(layout)
              }
          }
          FICHIER_EOF
          mkdir -p VeepeeBubble/app/src/main/java/fr/ced/bubble
          cat > VeepeeBubble/app/src/main/java/fr/ced/bubble/PriceAccessibilityService.kt <<'FILE_EOF'
          paquet fr.ced.bubble

          importer android.accessibilityservice.AccessibilityService
          importer android.content.ClipData
          importer android.content.ClipboardManager
          importer android.graphics.Color
          importer android.graphics.PixelFormat
          importer android.graphics.drawable.GradientDrawable
          importer android.os.Handler
          importer android.os.Looper
          importer android.view.Gravity
          importer android.view.WindowManager
          importer android.view.accessibility.AccessibilityEvent
          importer android.view.accessibility.AccessibilityNodeInfo
          importer android.widget.TextView
          importer android.widget.Toast

          classe PriceAccessibilityService : AccessibilityService() {

              private val handler = Handler(Looper.getMainLooper())
              private val scanRunnable = Runnable { scan() }
              var privée bubble: TextView? = null
              variable privée développée = vrai
              var shortText = ""
              var privée longText = ""
              var privée lastDump = ""

              remplacer fun onAccessibilityEvent(event: AccessibilityEvent?) {
                  val pkg = event?.packageName?.toString() ?: return
                  si (pkg == nom_du_package || pkg == "com.android.systemui") retourner
                  gestionnaire.supprimerCallbacks(scanRunnable)
                  handler.postDelayed(scanRunnable, 700)
              }

              remplacer fun onInterrupt() {}

              remplacer fun onDestroy() {
                  supprimerBulle()
                  super.onDestroy()
              }

              fonction privée scan() {
                  val racine = racineDansFenêtreActive ?: retourner
                  val pkg = root.packageName?.toString() ?: return
                  si (pkg == nom_du_package || pkg == "com.android.systemui") retourner

                  val filter = getSharedPreferences("cfg", MODE_PRIVATE).getString("filter", "")!!.trim()
                  si (filter.isNotEmpty() && !pkg.contains(filter, ignoreCase = true)) {
                      supprimerBulle()
                      retour
                  }

                  val textes = ArrayList<String>()
                  collecter(racine, textes, 0)
                  val r = ScreenParser.parse(texts)
                  lastDump = "package: $pkg\n" + texts.joinToString("\n")

                  si (textes.isEmpty()) {
                      shortText = "âš écran illisible"
                      longText = "Aucun texte lu dans $pkg.\nL'appli bloque peut-être l'accessibilité (plan B : capture d'écran)."
                  } sinon si (r.prices.isEmpty()) {
                      shortText = "Aucun prix détecté"
                      longText = "$pkg\n${texts.size} textes plus, pas de prix en â‚¬.\nAppui long = copier le texte lu."
                  } autre {
                      val fmt = { v: Double -> String.format(java.util.Locale.FRANCE, "%.2f \u20AC", v) }
                      val bas = r.prix.premier()
                      val haut = r.prix.dernier()
                      shortText = si (bas == haut) fmt(bas) sinon "${fmt(bas)} (barré ${fmt(haut)})"
                      longText = construireString {
                          ajouter(textecourt)
                          r.discountPercent?.let { append("\nRemise affiché : -$it %") }
                          append("\nProduit ? ${r.title ?: "non détecté"}")
                          ajouter("\nPrix : ${r.prices.joinToString(" · ") { fmt(it) }}")
                          append("\n[$pkg] appui long = copieur")
                      }
                  }
                  montrer()
              }

              fonction privée collect(n: AccessibilityNodeInfo?, out: MutableList<String>, depth: Int) {
                  si (n == null || profondeur > 40) retourner
                  n.text?.toString()?.trim()?.takeIf { it.isNotEmpty() }?.let { out.add(it) }
                  n.contentDescription?.toString()?.trim()?.takeIf { it.isNotEmpty() }?.let { out.add(it) }
                  pour (i dans 0 jusqu'à n.childCount) collecter(n.getChild(i), out, profondeur + 1)
              }

              spectacle privé fun() {
                  val wm = getSystemService(WINDOW_SERVICE) as WindowManager
                  si (bulle == null) {
                      val tv = TextView(this).apply {
                          définirCouleurTexte(Couleur.BLANC)
                          taille du texte = 13f
                          largeur maximale = 760
                          définirPadding(28, 18, 28, 18)
                          arrière-plan = GradientDrawable().apply {
                              rayon du coin = 40°
                              définirCouleur(Couleur.parseColor("#E6222222"))
                          }
                          setOnClickListener { expanded = !expanded; render() }
                          setOnLongClickListener {
                              val cm = getSystemService(CLIPBOARD_SERVICE) as ClipboardManager
                              cm.setPrimaryClip(ClipData.newPlainText("dump", lastDump))
                              Toast.makeText(context, "Texte de l'écran copie", Toast.LENGTH_SHORT).show()
                              vrai
                          }
                      }
                      val lp = WindowManager.LayoutParams(
                          WindowManager.LayoutParams.WRAP_CONTENT,
                          WindowManager.LayoutParams.WRAP_CONTENT,
                          WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
                          WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
                          PixelFormat.TRANSLUCENT
                      ).appliquer {
                          gravité = Gravité.HAUT ou Gravité.FIN
                          x = 16
                          y = 220
                      }
                      wm.addView(tv, lp)
                      bulle = télévision
                  }
                  rendre()
              }

              private fun render() {
                  bulle?.text = si (développé) longTexte sinon shortTexte
              }

              fonction privée supprimerBubble() {
                  bulle?.let {
                      essayer {
                          (getSystemService(WINDOW_SERVICE) as WindowManager).removeView(it)
                      } catch (_: Exception) {}
                  }
                  bulle = nulle
              }
          }
          FICHIER_EOF
          mkdir -p VeepeeBubble/app/src/main/java/fr/ced/bubble
          cat > VeepeeBubble/app/src/main/java/fr/ced/bubble/ScreenParser.kt <<'FILE_EOF'
          paquet fr.ced.bubble

          classe de données ParsedScreen(
              val titre : Chaîne ?,
              prix val : List<Double>,
              val discountPercent: Int?
          )

          objet ScreenParser {
              // Ex : "49â‚¬", "49,90 â‚¬", "1 299,00 â‚¬"
              private val priceRx = Regex("""(\d{1,3}(?:[\u00a0 .]\d{3})+|\d+)(?:[.,](\d{1,2}))?\s?""" + "\u20AC")
              valeur privée discountRx = Regex("""-\s?(\d{1,2})\s?%""")

              fonction parse(texts: List<String>): ParsedScreen {
                  val prix = mutableSetOf<Double>()
                  var discount: Int? = null
                  pour (t dans les textes) {
                      pour (m dans priceRx.findAll(t)) {
                          val intPart = m.groupValues[1].filter { it.isDigit() }
                          val dec = m.groupValues[2].ifEmpty { "0" }
                          val v = "$intPart.$dec".toDoubleOrNull()
                          si (v != null && v > 0.0) prices.add(v)
                      }
                      discountRx.find(t)?.let { d ->
                          val p = d.groupValues[1].toInt()
                          si (remise == null || p > remise!!) remise = p
                      }
                  }
                  // Titre candidat : ​​texte le plus long (15-120 car.) sans prix
                  val titre = textes
                      .filter { it.length dans 15..120 && !it.contains("\u20AC") }
                      .maxByOrNull { it.length }
                  renvoie ParsedScreen(titre, prix.sorted(), réduction)
              }
          }
          FICHIER_EOF
          mkdir -p VeepeeBubble/app/src/main/res/values
          cat > VeepeeBubble/app/src/main/res/values/strings.xml <<'FILE_EOF'
          <ressources>
              <string name="app_name">Bulle Prix</string>
              <string name="service_desc">Lit le titre et le prix affichés à l'écran pour les comparer.</string>
          </ressources>
          FICHIER_EOF
          mkdir -p VeepeeBubble/app/src/main/res/xml
          cat > VeepeeBubble/app/src/main/res/xml/accessibility_config.xml <<'FILE_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
              android:accessibilityEventTypes="typeWindowContentChanged|typeWindowStateChanged|typeViewScrolled"
              android:accessibilityFeedbackType="feedbackGeneric"
              android:accessibilityFlags="flagIncludeNotImportantViews"
              android:canRetrieveWindowContent="true"
              android:notificationTimeout="300"
              android:description="@string/service_desc" />
          FICHIER_EOF
          mkdir -p VeepeeBubble
          cat > VeepeeBubble/build.gradle.kts <<'FILE_EOF'
          plugins {
              id("com.android.application") version "8.5.2" appliquer faux
              id("org.jetbrains.kotlin.android") version "1.9.24" appliquer faux
          }
          FICHIER_EOF
          mkdir -p VeepeeBubble
          chat > ​​VeepeeBubble/gradle.properties <<'FILE_EOF'
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          kotlin.code.style=officiel
          FICHIER_EOF
          mkdir -p VeepeeBubble
          chat > ​​VeepeeBubble/settings.gradle.kts <<'FILE_EOF'
          Gestion des plugins { dépôts { google(); mavenCentral(); gradlePluginPortal() } }
          Gestion des résolutions de dépendances { dépôts { google(); mavenCentral() } }
          rootProject.name = "VeepeeBubble"
          inclure(":app")
          FICHIER_EOF
      - nom : Générer l’APK de débogage
        répertoire de travail : VeepeeBubble
        Exécuter : gradle assembleDebug --no-daemon
      - nom : Publier l'APK
        utilise : softprops/action-gh-release@v2
        avec:
          tag_name: build-${{ github.run_number }}
          nom : Bulle Prix build ${{ github.run_number }}
          Fichiers : VeepeeBubble/app/build/outputs/apk/debug/app-debug.apk
