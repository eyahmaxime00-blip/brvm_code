import os
import json
from paddleocr import PaddleOCR
from pdf2image import convert_from_path

# 1. Initialisation du moteur PaddleOCR (français/anglais, mode détection de tableaux)
ocr = PaddleOCR(use_angle_cls=True, lang='fr', show_log=False)

DOSSIER_PDF = "etats_financiers_brvm"
DOSSIER_SORTIE = "output_parsed"
os.makedirs(DOSSIER_SORTIE, exist_ok=True)

def extraire_donnees_pdf(chemin_pdf):
    """
    Convertit un PDF en images et extrait le texte structuré avec PaddleOCR.
    """
    print(f"📄 Analyse en cours : {os.path.basename(chemin_pdf)}...")
    
    # Conversion du PDF en images (1 image par page)
    # Note: Vous pouvez cibler uniquement les pages clés (ex: pages 5 à 25)
    images = convert_from_path(chemin_pdf, first_page=1, last_page=15)
    
    resultats_document = []

    for idx, image in enumerate(images):
        chemin_temp_image = f"temp_page_{idx+1}.png"
        image.save(chemin_temp_image, "PNG")
        
        # Exécution de l'OCR sur la page
        resultat = ocr.ocr(chemin_temp_image, cls=True)
        
        lignes_page = []
        if resultat and resultat[0]:
            for ligne in resultat[0]:
                texte = ligne[1][0]       # Texte extrait
                confiance = ligne[1][1]   # Score de confiance
                lignes_page.append({"texte": texte, "confiance": round(confiance, 2)})
        
        resultats_document.append({
            "page": idx + 1,
            "contenu": lignes_page
        })
        
        # Nettoyage de l'image temporaire
        if os.path.exists(chemin_temp_image):
            os.remove(chemin_temp_image)

    return resultats_document

def executer_parsing():
    all_parsed_data = {}
    
    # Parcourt tous les PDF téléchargés par la Tool 1
    for racine, _, fichiers in os.walk(DOSSIER_PDF):
        for fichier in fichiers:
            if fichier.endswith(".pdf"):
                chemin_complet = os.path.join(racine, fichier)
                donnees_extraites = extraire_donnees_pdf(chemin_complet)
                all_parsed_data[fichier] = donnees_extraites

    # Sauvegarde du résultat structuré en JSON
    fichier_json = os.path.join(DOSSIER_SORTIE, "etats_financiers_parsed.json")
    with open(fichier_json, "w", encoding="utf-8") as f:
        json.dump(all_parsed_data, f, ensure_ascii=False, indent=4)
        
    print(f"\n✅ Extraction terminée ! Résultat sauvegardé dans : {fichier_json}")

if __name__ == "__main__":
    executer_parsing()
