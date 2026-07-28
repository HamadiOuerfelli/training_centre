# Guide réutilisable : Mode clair/sombre + Multilangue (HTML/CSS/JS vanilla)

Ce guide documente le système de **thème clair/sombre** et de **multilangue (i18n)** utilisé dans `index.html`. Il est conçu pour être copié-collé dans n'importe quel projet HTML/CSS/JS statique (sans framework), en n'adaptant que les valeurs (couleurs, textes).

Aucune dépendance externe n'est nécessaire : tout fonctionne avec du HTML, CSS et JavaScript natifs.

---

## Sommaire

1. [Vue d'ensemble](#vue-densemble)
2. [Partie 1 — Script anti-flash (FOUC)](#partie-1--script-anti-flash-fouc)
3. [Partie 2 — Variables CSS à deux thèmes](#partie-2--variables-css-à-deux-thèmes)
4. [Partie 3 — Bouton de bascule du thème](#partie-3--bouton-de-bascule-du-thème)
5. [Partie 4 — Attributs `data-i18n` dans le HTML](#partie-4--attributs-data-i18n-dans-le-html)
6. [Partie 5 — Dictionnaire de traduction et fonction d'application](#partie-5--dictionnaire-de-traduction-et-fonction-dapplication)
7. [Checklist d'intégration dans un nouveau projet](#checklist-dintégration-dans-un-nouveau-projet)
8. [Pièges fréquents](#pièges-fréquents)
9. [Exemple minimal complet](#exemple-minimal-complet)

---

## Vue d'ensemble

Le système repose sur deux mécanismes indépendants qui peuvent être utilisés séparément ou ensemble :

| Fonctionnalité | Mécanisme | Persistance |
|---|---|---|
| Thème clair/sombre | Attribut `data-theme` sur `<html>` + variables CSS | `localStorage.theme` |
| Multilangue (FR/EN...) | Attributs `data-i18n*` sur les éléments + dictionnaire JS | `localStorage.lang` |

Les deux préférences sont lues **avant** le premier rendu de la page (dans un `<script>` placé tout en haut du `<head>`) pour éviter un flash visuel (mauvaise couleur ou mauvaise langue affichée une fraction de seconde avant correction).

---

## Partie 1 — Script anti-flash (FOUC)

À placer **dans `<head>`, avant les feuilles de style et avant tout autre script** :

```html
<script>
(function(){
  try{
    var t = localStorage.getItem('theme');
    if(!t){ t = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'; }
    document.documentElement.setAttribute('data-theme', t);

    var l = localStorage.getItem('lang') || 'fr';
    document.documentElement.setAttribute('lang', l);
  }catch(e){}
})();
</script>
```

**Pourquoi ça marche :** ce script est synchrone et s'exécute avant que le navigateur ne peigne la page. Il pose l'attribut `data-theme="dark"` ou `data-theme="light"` sur `<html>` immédiatement, donc le CSS (qui cible cet attribut) applique la bonne palette dès le premier rendu — sans "flash blanc" avant de passer en sombre.

**Pourquoi `try/catch` :** `localStorage` peut lever une exception dans certains contextes restreints (mode privé strict de certains navigateurs, iframes sandboxées). Le `try/catch` garantit que la page continue de se charger même dans ce cas, avec un thème par défaut.

**Détection de préférence système :** `window.matchMedia('(prefers-color-scheme: dark)')` respecte le thème OS de l'utilisateur si aucune préférence n'a encore été enregistrée manuellement.

---

## Partie 2 — Variables CSS à deux thèmes

Définir **toutes** les couleurs comme variables CSS dans `:root`, puis les redéfinir dans un bloc `:root[data-theme="dark"]` :

```css
:root{
  --ink:#0f1229;           /* texte principal */
  --ink-soft:#4b4f6b;      /* texte secondaire */
  --bg:#f7f8fc;            /* fond de page */
  --bg-alt:#f1f2fb;        /* fond de section alternée */
  --card:#ffffff;          /* fond des cartes */
  --line:#e7e9f5;          /* bordures */
  --header-bg:rgba(247,248,252,.75); /* header au scroll (avec transparence) */
  --shadow-sm: 0 2px 10px rgba(30,32,80,.06);
  --shadow-md: 0 10px 30px rgba(30,32,80,.10);
  --shadow-lg: 0 25px 60px rgba(30,32,80,.16);
}

:root[data-theme="dark"]{
  --ink:#eef0fb;
  --ink-soft:#a2a5c9;
  --bg:#0c0e1f;
  --bg-alt:#11142b;
  --card:#161933;
  --line:#272b52;
  --header-bg:rgba(12,14,31,.75);
  --shadow-sm: 0 2px 10px rgba(0,0,0,.35);
  --shadow-md: 0 10px 30px rgba(0,0,0,.45);
  --shadow-lg: 0 25px 60px rgba(0,0,0,.6);
}
```

**Règle d'or : ne jamais écrire de couleur en dur (`#fff`, `background:white`, etc.) dans le CSS des composants.** Tout doit passer par `var(--xxx)`. Si une couleur est codée en dur quelque part, elle restera figée dans un thème et cassera visuellement l'autre.

**Exception valable :** les sections déjà volontairement sombres dans les deux thèmes (bandeaux de stats, footer, blocs "contact" à fond dégradé sombre) peuvent garder des couleurs fixes — elles ne sont pas censées changer avec le thème, elles font partie de l'identité visuelle.

**Transition douce (optionnel mais recommandé) :**
```css
body{ transition: background .4s ease, color .4s ease; }
.card, .price-card, .faq-item{ transition: background .4s ease, /* + vos autres transitions */; }
```

---

## Partie 3 — Bouton de bascule du thème

**HTML** (un simple bouton, placé où vous voulez dans le header) :
```html
<button class="icon-toggle" id="themeToggle" aria-label="Toggle dark mode" title="Toggle dark mode">🌙</button>
```

**CSS minimal :**
```css
.icon-toggle{
  display:flex;align-items:center;justify-content:center;
  width:42px;height:42px;border-radius:50%;
  background:rgba(91,94,244,.08);
  border:1px solid rgba(91,94,244,.18);
  transition:all .3s ease;
}
.icon-toggle:hover{ background:rgba(91,94,244,.16); transform:translateY(-2px) rotate(-6deg); }
```

**JavaScript :**
```js
const themeToggle = document.getElementById('themeToggle');

function applyTheme(theme){
  document.documentElement.setAttribute('data-theme', theme);
  themeToggle.textContent = theme === 'dark' ? '☀️' : '🌙'; // icône = action possible, pas l'état actuel
  localStorage.setItem('theme', theme);
}

// Synchronise l'icône avec le thème déjà posé par le script anti-flash
applyTheme(document.documentElement.getAttribute('data-theme') || 'light');

themeToggle.addEventListener('click', () => {
  const next = document.documentElement.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
  applyTheme(next);
});
```

**Détail important :** l'icône affichée est celle de l'action à venir (🌙 = "cliquez pour passer en sombre", ☀️ = "cliquez pour passer en clair"), pas une icône représentant l'état actuel. C'est la convention la plus intuitive pour l'utilisateur.

---

## Partie 4 — Attributs `data-i18n` dans le HTML

Trois attributs, selon le type de contenu à traduire :

| Attribut | Usage | Exemple |
|---|---|---|
| `data-i18n` | Texte simple (remplace `textContent`) | `<h2 data-i18n="title_key">Titre par défaut</h2>` |
| `data-i18n-placeholder` | Placeholder d'un champ de formulaire | `<input data-i18n-placeholder="ph_key" placeholder="...">` |
| `data-i18n-html` | Contenu HTML riche (remplace `innerHTML`) | `<h1 data-i18n-html="hero_key">Texte <span class="grad-text">stylé</span></h1>` |

**Pourquoi 3 variantes et pas une seule ?**
- `textContent` est plus sûr (pas de risque d'injection HTML) → à utiliser par défaut pour 90% des textes.
- `data-i18n-html` n'est nécessaire que lorsque le texte contient une balise imbriquée (ex : un `<span>` avec un dégradé de couleur au milieu d'un titre). Ne l'utiliser que dans ce cas précis, avec du contenu que vous contrôlez vous-même (jamais de texte venant d'un utilisateur).
- `data-i18n-placeholder` cible un attribut, pas un contenu — `textContent` ne fonctionnerait pas sur un `<input>`.

**Le texte présent par défaut dans le HTML doit être la langue principale du site** (ex. français) — c'est ce qui s'affiche si le JavaScript est désactivé ou avant l'exécution du script de traduction, et ça garde le HTML lisible/éditable directement.

---

## Partie 5 — Dictionnaire de traduction et fonction d'application

**Structure du dictionnaire** — un objet avec une clé par langue, contenant les mêmes clés que dans le HTML :

```js
const translations = {
  fr: {
    title_key: "Titre par défaut",
    hero_key: 'Texte <span class="grad-text">stylé</span>',
    ph_key: "Votre nom",
  },
  en: {
    title_key: "Default title",
    hero_key: 'Text <span class="grad-text">styled</span>',
    ph_key: "Your name",
  }
};
```

**Fonction d'application** — parcourt le DOM et remplace le contenu selon la langue active :

```js
const langToggle = document.getElementById('langToggle');
let currentLang = localStorage.getItem('lang') || 'fr';

function applyTranslations(lang){
  const dict = translations[lang];
  document.documentElement.setAttribute('lang', lang);

  document.querySelectorAll('[data-i18n]').forEach(el => {
    const key = el.getAttribute('data-i18n');
    if (dict[key] !== undefined) el.textContent = dict[key];
  });

  document.querySelectorAll('[data-i18n-html]').forEach(el => {
    const key = el.getAttribute('data-i18n-html');
    if (dict[key] !== undefined) el.innerHTML = dict[key];
  });

  document.querySelectorAll('[data-i18n-placeholder]').forEach(el => {
    const key = el.getAttribute('data-i18n-placeholder');
    if (dict[key] !== undefined) el.setAttribute('placeholder', dict[key]);
  });

  langToggle.textContent = lang === 'fr' ? 'EN' : 'FR'; // affiche la langue vers laquelle basculer
  localStorage.setItem('lang', lang);
}

applyTranslations(currentLang);

langToggle.addEventListener('click', () => {
  currentLang = currentLang === 'fr' ? 'en' : 'fr';
  applyTranslations(currentLang);
});
```

**Point d'attention — éléments à hauteur dynamique (accordéons FAQ, menus déroulants animés) :** si un élément était ouvert avec une hauteur calculée en JS (`max-height: Xpx`) au moment du changement de langue, et que le nouveau texte est plus long ou plus court, cette hauteur devient incorrecte. Il faut la recalculer après traduction :

```js
document.querySelectorAll('.faq-item.open .faq-a').forEach(a => {
  a.style.maxHeight = a.scrollHeight + 'px';
});
```
(cette ligne doit être ajoutée à la fin de `applyTranslations`)

---

## Checklist d'intégration dans un nouveau projet

1. Coller le script anti-flash en tout début de `<head>`.
2. Définir les variables `:root { ... }` (thème clair) et `:root[data-theme="dark"] { ... }` (thème sombre).
3. Remplacer toute couleur codée en dur dans le CSS des composants par une `var(--xxx)`.
4. Ajouter le bouton `#themeToggle` dans le header + son CSS `.icon-toggle`.
5. Ajouter le bouton `#langToggle` juste à côté (même style `.icon-toggle`).
6. Sur chaque texte visible du HTML, ajouter `data-i18n="cle_unique"` (ou `-html` / `-placeholder` selon le cas).
7. Construire l'objet `translations` avec une entrée par langue et toutes les clés utilisées dans le HTML.
8. Copier les fonctions `applyTheme()` et `applyTranslations()` + leurs écouteurs de clic, en bas de page avant `</body>`.
9. Tester : recharger la page après avoir changé de thème/langue → la préférence doit être conservée (grâce à `localStorage`).
10. Tester en réduisant la fenêtre (mobile) : les boutons `.icon-toggle` doivent rester visibles (ne pas les masquer dans les media queries qui cachent le menu desktop).

## Pièges fréquents

- **Couleur oubliée en dur** → un composant reste identique en clair/sombre. Chercher toutes les occurrences de codes hexadécimaux (`#fff`, `#000`, etc.) et `rgba(255,255,255` / `rgba(0,0,0` dans le CSS des composants (pas des dégradés décoratifs volontairement sombres) et les remplacer par des variables.
- **Clé de traduction manquante dans une langue** → `dict[key]` vaut `undefined`, la fonction ignore silencieusement l'élément (il garde son ancien texte). Toujours garder les deux objets de langue synchronisés — même nombre de clés.
- **`data-i18n-html` sur du contenu utilisateur** → risque d'injection si jamais une clé venait à contenir du texte non maîtrisé (ex. saisi via un formulaire d'admin sans échappement). Réserver cet attribut au contenu que vous écrivez vous-même en dur dans le dictionnaire.
- **Bouton de thème cliqué avant que le DOM soit prêt** → toujours placer le `<script>` de logique juste avant `</body>`, après tous les éléments HTML qu'il cible.
- **Oubli de recalculer les `max-height` des accordéons ouverts après un changement de langue** → contenu tronqué ou espace vide en bas d'un panneau ouvert.

---

## Exemple minimal complet

```html
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<script>
(function(){
  try{
    var t = localStorage.getItem('theme') || (matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
    document.documentElement.setAttribute('data-theme', t);
    document.documentElement.setAttribute('lang', localStorage.getItem('lang') || 'fr');
  }catch(e){}
})();
</script>
<style>
  :root{ --bg:#fff; --ink:#111; }
  :root[data-theme="dark"]{ --bg:#111; --ink:#eee; }
  body{ background:var(--bg); color:var(--ink); font-family:sans-serif; transition:.3s; }
</style>
</head>
<body>
  <button id="themeToggle">🌙</button>
  <button id="langToggle">EN</button>
  <h1 data-i18n="title">Bonjour</h1>

  <script>
    const translations = { fr:{ title:"Bonjour" }, en:{ title:"Hello" } };
    let currentLang = localStorage.getItem('lang') || 'fr';
    const langToggle = document.getElementById('langToggle');

    function applyTranslations(lang){
      document.documentElement.setAttribute('lang', lang);
      document.querySelectorAll('[data-i18n]').forEach(el => el.textContent = translations[lang][el.dataset.i18n]);
      langToggle.textContent = lang === 'fr' ? 'EN' : 'FR';
      localStorage.setItem('lang', lang);
    }
    applyTranslations(currentLang);
    langToggle.addEventListener('click', () => { currentLang = currentLang === 'fr' ? 'en' : 'fr'; applyTranslations(currentLang); });

    const themeToggle = document.getElementById('themeToggle');
    function applyTheme(theme){
      document.documentElement.setAttribute('data-theme', theme);
      themeToggle.textContent = theme === 'dark' ? '☀️' : '🌙';
      localStorage.setItem('theme', theme);
    }
    applyTheme(document.documentElement.getAttribute('data-theme'));
    themeToggle.addEventListener('click', () => {
      applyTheme(document.documentElement.getAttribute('data-theme') === 'dark' ? 'light' : 'dark');
    });
  </script>
</body>
</html>
```

Cet exemple de ~40 lignes contient toute la mécanique ; il suffit de l'étoffer avec vos propres couleurs, clés et contenus.

---

*Implémentation complète et fonctionnelle : voir [`index.html`](./index.html) dans ce dépôt.*
