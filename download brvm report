import os
import re
import csv
import time
import unicodedata
from datetime import datetime, timedelta
from urllib.parse import urljoin, urlparse, unquote

from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeoutError


# ==============================================================
# CONFIGURATION & CONSTANTES
# ==============================================================

BASE_URL = "https://www.brvm.org"

# URLs Source
URL_LISTE_SOCIETES = f"{BASE_URL}/fr/rapports-societes-cotees"
URL_ANNONCES_EMETTEURS = f"{BASE_URL}/fr/emetteurs/type-annonces/convocations-assemblees-generales"
URL_BULLETINS_COTE = f"{BASE_URL}/fr/bulletins-officiels-de-la-cote"
URL_RESUME = f"{BASE_URL}/fr/resume"

# Arborescence des Dossiers
DOSSIER_DATA = "data"
DOSSIER_DOCUMENTS = os.path.join(DOSSIER_DATA, "documents")       # Stockage PERMANENT
DOSSIER_TEMPORAIRE = os.path.join(DOSSIER_DATA, "temporary")       # Stockage TEMPORAIRE (24h)

DOSSIER_ANNONCES = os.path.join(DOSSIER_TEMPORAIRE, "annonces_emetteurs")
DOSSIER_BULLETINS = os.path.join(DOSSIER_TEMPORAIRE, "bulletins_cote")
DOSSIER_RESUMES = os.path.join(DOSSIER_TEMPORAIRE, "resumes_cotation")

# Catalogues CSV
CATALOGUE_CSV = os.path.join(DOSSIER_DOCUMENTS, "catalogue_documents.csv")
CATALOGUE_TEMPORAIRE_CSV = os.path.join(DOSSIER_TEMPORAIRE, "temporary_catalogue.csv")

# Paramètres Exécution
NB_PAGES_SOCIETES = 1        # Définir None pour traiter toutes les pages
HEADLESS = True              # Mode sans interface graphique
DELAI_ENTRE_REQUETES = 0.5   # Pause de courtoisie en secondes
DUREE_TEMPORAIRE_HEURES = 24 # Rétention de la mémoire vive / temporaire

# Catégories Financières Ciblées
CATEGORIES_CIBLES = {
    "etats financiers": "etats_financiers",
    "rapports annuels": "rapports_annuels",
    "rapports semestriels": "rapports_semestriels",
    "rapports trimestriels": "rapports_trimestriels",
    "commentaires sur l activite": "commentaires_activite",
}


# ==============================================================
# OUTILS DE NETTOYAGE ET FORMATAGE
# ==============================================================

def enlever_accents(texte):
    if not texte:
        return ""
    texte = unicodedata.normalize("NFD", texte)
    return "".join(c for c in texte if unicodedata.category(c) != "Mn")

def normaliser_texte(texte):
    if not texte:
        return ""
    texte = enlever_accents(texte).lower().replace("’", "'")
    return re.sub(r"\s+", " ", texte).strip()

def nettoyer_nom(nom):
    if not nom:
        return "document"
    nom = enlever_accents(nom)
    nom = re.sub(r'[\\/*?:"<>|]', "", nom)
    nom = re.sub(r"\s+", "_", nom)
    nom = re.sub(r"_+", "_", nom)
    return nom.strip("_. ")[:180]

def url_complete(href):
    return urljoin(BASE_URL, href) if href else None

def date_maintenant():
    return datetime.now()

def dossier_date():
    return date_maintenant().strftime("%Y-%m-%d")

def fichier_date_heure():
    return date_maintenant().strftime("%Y%m%d_%H%M%S")


# ==============================================================
# GESTION DES DOSSIERS ET CATALOGUES
# ==============================================================

def initialiser_dossiers():
    for dossier in [DOSSIER_DOCUMENTS, DOSSIER_TEMPORAIRE, DOSSIER_ANNONCES, DOSSIER_BULLETINS, DOSSIER_RESUMES]:
        os.makedirs(dossier, exist_ok=True)

def initialiser_catalogues():
    initialiser_dossiers()
    
    # Catalogue Permanent
    if not os.path.exists(CATALOGUE_CSV):
        with open(CATALOGUE_CSV, "w", newline="", encoding="utf-8-sig") as f:
            csv.DictWriter(f, fieldnames=[
                "entreprise", "categorie", "nom_document", "fichier", "chemin", "url_source", "url_page_categorie", "statut"
            ]).writeheader()

    # Catalogue Temporaire
    if not os.path.exists(CATALOGUE_TEMPORAIRE_CSV):
        with open(CATALOGUE_TEMPORAIRE_CSV, "w", newline="", encoding="utf-8-sig") as f:
            csv.DictWriter(f, fieldnames=[
                "type", "date_collecte", "expiration", "titre", "fichier", "chemin", "url_source", "statut"
            ]).writeheader()

def ajouter_catalogue_permanent(entreprise, categorie, nom_document, fichier, chemin, url_source, url_page, statut):
    with open(CATALOGUE_CSV, "a", newline="", encoding="utf-8-sig") as f:
        csv.DictWriter(f, fieldnames=[
            "entreprise", "categorie", "nom_document", "fichier", "chemin", "url_source", "url_page_categorie", "statut"
        ]).writerow({
            "entreprise": entreprise, "categorie": categorie, "nom_document": nom_document,
            "fichier": fichier, "chemin": chemin, "url_source": url_source,
            "url_page_categorie": url_page, "statut": statut
        })

def ajouter_catalogue_temporaire(type_doc, titre, fichier, chemin, url_source, statut):
    maintenant = date_maintenant()
    expiration = maintenant + timedelta(hours=DUREE_TEMPORAIRE_HEURES)
    with open(CATALOGUE_TEMPORAIRE_CSV, "a", newline="", encoding="utf-8-sig") as f:
        csv.DictWriter(f, fieldnames=[
            "type", "date_collecte", "expiration", "titre", "fichier", "chemin", "url_source", "statut"
        ]).writerow({
            "type": type_doc, "date_collecte": maintenant.isoformat(), "expiration": expiration.isoformat(),
            "titre": titre, "fichier": fichier, "chemin": chemin, "url_source": url_source, "statut": statut
        })

def nettoyer_memoire_temporaire():
    """Supprime automatiquement les fichiers temporaires de plus de 24h."""
    print("\n🧹 NETTOYAGE DE LA MÉMOIRE TEMPORAIRE (> 24H)...")
    if not os.path.exists(DOSSIER_TEMPORAIRE):
        return

    maintenant = date_maintenant()
    supprimes = 0

    for racine, _, fichiers in os.walk(DOSSIER_TEMPORAIRE, topdown=False):
        for file in fichiers:
            chemin = os.path.join(racine, file)
            if chemin == CATALOGUE_TEMPORAIRE_CSV:
                continue
            try:
                age = maintenant - datetime.fromtimestamp(os.path.getmtime(chemin))
                if age >= timedelta(hours=DUREE_TEMPORAIRE_HEURES):
                    os.remove(chemin)
                    supprimes += 1
                    print(f"   🗑️ Supprimé : {chemin}")
            except Exception as e:
                print(f"   ⚠️ Erreur suppression {chemin}: {e}")

    print(f"   ✅ Total : {supprimes} fichier(s) temporaire(s) nettoyé(s).")


# ==============================================================
# SCRAPING ET TÉLÉCHARGEMENT
# ==============================================================

def telecharger_url(context, url, chemin_final):
    """Télécharge un fichier binaire (PDF/HTML/TXT) de manière sécurisée."""
    try:
        response = context.request.get(url, timeout=60000)
        if response.status != 200:
            return False, f"erreur_http_{response.status}"
        
        contenu = response.body()
        if not contenu:
            return False, "fichier_vide"

        with open(chemin_final, "wb") as f:
            f.write(contenu)
        return True, "telecharge"
    except Exception as e:
        return False, f"erreur: {e}"

def scraper_rapports_financiers(context, stats):
    """Extraction et téléchargement PERMANENT des états financiers."""
    print("\n" + "="*70 + "\n📚 RAPPORTS ET ÉTATS FINANCIERS (STOCKAGE PERMANENT)\n" + "="*70)
    page = context.new_page()

    try:
        page.goto(URL_LISTE_SOCIETES, wait_until="domcontentloaded", timeout=60000)
        time.sleep(1)

        # Extraction des liens de sociétés
        liens = page.locator("td a").all()
        societes = []
        for l in liens:
            href = l.get_attribute("href")
            txt = l.inner_text().strip()
            if href and "/fr/rapports-societe-cotes/" in href:
                url_c = url_complete(href)
                if not any(u == url_c for _, u in societes):
                    societes.append((txt, url_c))

        print(f"🏢 {len(societes)} société(s) détectée(s) sur la page.")

        for idx, (nom_soc, url_soc) in enumerate(societes, start=1):
            stats["societes"] += 1
            print(f"\n[{idx}/{len(societes)}] 🏢 Société : {nom_soc}")
            
            page_soc = context.new_page()
            try:
                page_soc.goto(url_soc, wait_until="domcontentloaded", timeout=60000)
                
                # Clic ou détection sur la catégorie "États Financiers"
                liens_cat = page_soc.locator("a").all()
                for l_cat in liens_cat:
                    txt_cat = normaliser_texte(l_cat.inner_text())
                    if "etat financier" in txt_cat:
                        url_cat = url_complete(l_cat.get_attribute("href"))
                        print(f"   📂 Traitement des États Financiers : {url_cat}")
                        
                        # Traitement de la page de la catégorie
                        page_cat = context.new_page()
                        page_cat.goto(url_cat, wait_until="domcontentloaded")
                        
                        pdf_liens = page_cat.locator("a[href*='.pdf']").all()
                        for pdf_idx, pdf_l in enumerate(pdf_liens, start=1):
                            pdf_url = url_complete(pdf_l.get_attribute("href"))
                            nom_doc = pdf_l.inner_text().strip() or f"etat_financier_{pdf_idx}"
                            
                            # Préparation du dossier
                            dossier_target = os.path.join(DOSSIER_DOCUMENTS, nettoyer_nom(nom_soc), "etats_financiers")
                            os.makedirs(dossier_target, exist_ok=True)
                            
                            filename = f"{nettoyer_nom(nom_doc)}.pdf"
                            chemin_final = os.path.join(dossier_target, filename)
                            
                            if os.path.exists(chemin_final):
                                print(f"      ⏭️ Déjà présent : {filename}")
                                stats["deja_presents"] += 1
                                statut = "deja_present"
                            else:
                                succes, statut = telecharger_url(context, pdf_url, chemin_final)
                                if succes:
                                    print(f"      ✅ Téléchargé : {filename}")
                                    stats["telecharges"] += 1
                                else:
                                    print(f"      ❌ Erreur {filename} : {statut}")
                                    stats["erreurs"] += 1

                            ajouter_catalogue_permanent(
                                entreprise=nom_soc, categorie="États Financiers",
                                nom_document=nom_doc, fichier=filename, chemin=chemin_final,
                                url_source=pdf_url, url_page=url_cat, statut=statut
                            )
                        page_cat.close()
            except Exception as e:
                print(f"   ⚠️ Erreur traitement {nom_soc} : {e}")
            finally:
                page_soc.close()
    finally:
        page.close()


# ==============================================================
# PIPELINE PRINCIPAL
# ==============================================================

def executer_pipeline_brvm():
    print("\n" + "="*70 + "\n🚀 FINANCIAL AGENT — BRVM AUTOMATED DOWNLOADER\n" + "="*70)
    
    initialiser_catalogues()
    nettoyer_memoire_temporaire()

    stats = {"societes": 0, "telecharges": 0, "deja_presents": 0, "erreurs": 0}

    with sync_playwright() as p:
        browser = p.chromium.launch(headless=HEADLESS)
        context = browser.new_context(user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0")

        try:
            # 1. Scraping des États Financiers Permanents
            scraper_rapports_financiers(context, stats)

            # Note : Les scrapers d'annonces, bulletins et résumés peuvent être chaînés ici
            # de la même manière pour alimenter le dossier 'temporary' avec expiration 24h.

        finally:
            browser.close()

    print("\n" + "="*70 + "\n🎉 PIPELINE BRVM EXÉCUTÉ AVEC SUCCÈS\n" + "="*70)
    print(f"   🏢 Sociétés scannées : {stats['societes']}")
    print(f"   ⬇️ Nouveaux PDF téléchargés : {stats['telecharges']}")
    print(f"   ⏭️ Fichiers déjà existants : {stats['deja_presents']}")
    print(f"   ❌ Erreurs de téléchargement : {stats['erreurs']}")
    print(f"   📋 Catalogue permanent mis à jour : {os.path.abspath(CATALOGUE_CSV)}")

if __name__ == "__main__":
    executer_pipeline_brvm()
