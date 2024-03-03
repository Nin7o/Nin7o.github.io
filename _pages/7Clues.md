---
title: "7Clues"
permalink: /7Clues/
author_profile: false
redirect_from:
  - /7Clues
---

Download an archive of the project : [Click here](/files/7Clues.zip)

Made with Eclipse with a team of 4 people. My first team project at ENSEEIHT.

Our project presentation : <a href="/files/7Clues.pdf" download>Click here to download.</a>

<object data="https://nin7o.github.io/files/7Clues.pdf" type="application/pdf" width=1920px height=1080px>
    <embed src="https://nin7o.github.io/files/7Clues.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="https://nin7o.github.io/files/7Clues.pdf">Download PDF</a>.</p>
    </embed>
</object>


I was in charge of implementing the run() method, which was called to update game graphics, NPC positions, etc... 
I also did a lot of work around collisions with the map, and coherent movement.

Draw()
==
```java
	@Override
	public void run() {

		double drawInterval = 1000000000 / FPS;
		double delta = 0;
		long lastTime = System.nanoTime();
		long currentTime;

		long timer = 0;
		int drawCount = 0;

		while (gameThread != null) {
			
			
			
			//System.out.println(bpause);
			// Pause
			if (keyH.isKeyTyped(KeyEvent.VK_P)) {
				bpause = !bpause;
				if (bpause) {

					

					this.getPlayer().addRichesse(1, 1);
					Main.setState(STATE.PAUSE);
				} else {
					bpause = false;
					Main.setState(STATE.GAME);
					
				}
				keyH.traiterTyped(KeyEvent.VK_P);
				
			}
			
			if (keyH.isKeyTyped(KeyEvent.VK_I)) {
				bsac = !bsac;
				if (bsac) {
					
					Main.setState(STATE.SAC);
				} else {
					bsac = false;
					Main.setState(STATE.GAME);
					
				}
				keyH.traiterTyped(KeyEvent.VK_I);
				
			}
			
			if (freeze) {
				if (keyH.isKeyTyped(KeyEvent.VK_ENTER)) {
					keyH.traiterTyped(KeyEvent.VK_ENTER);
					freeze = false;
					this.bdialogue = false;
					this.pnjParlant.desactiverDialogueActif();
					
				}
			}
			
			if (!bpause && !bsac && !freeze) {
				currentTime = System.nanoTime();

				delta += (currentTime - lastTime) / drawInterval;

				timer += (currentTime - lastTime);

				lastTime = currentTime;

				if (delta >= 10) {

					repaint();

					delta--;

					drawCount++;
					
					

				}

				if (timer >= 1000000000) {
					timer = 0;
					drawCount = 0;

				}		
			}

			
		}

	}
```

Collider object :
==
```java
package entite;

import java.util.ArrayList;
import java.util.List;

import gameWindow.*;
import map.*;

public class Collider {

	GamePanel gp;
	List<String> listeCollider;

	public Collider(GamePanel gp) {
		this.gp = gp;
		initCollisionsEntite();
	}
	
	private void initCollisionsEntite() {
        this.listeCollider = new ArrayList<>();
        
        // Initialisation de la liste avec des chaînes prédéfinies
        listeCollider.add("Porte1F");
        listeCollider.add("Porte2F");
        listeCollider.add("Biblio");
        listeCollider.add("Herbe");
        listeCollider.add("HerbeSol");
        
        // Affichage des éléments de la liste
        for (String str : listeCollider) {
            System.out.println(str);
        }
    }
	
	/* Vérifie si l'entite doit se téléporter ou non et réalise l'action si oui */
	public void checkTeleportation(EntiteMobile entite) {
		int teleportationOffSet = GamePanel.tileSize/2;
		int rowTeleportation = (entite.y + teleportationOffSet)/GamePanel.tileSize;
		int columnTeleportation = (entite.x + teleportationOffSet)/GamePanel.tileSize;
		
		Walls walls = gp.getCurrentWalls();
		
		int tile1 = walls.matrix[rowTeleportation][columnTeleportation];
		if  (!(rowTeleportation < 0 || rowTeleportation >= GamePanel.maxScreenRow || columnTeleportation >= GamePanel.maxScreenColumn
					|| columnTeleportation < 0)) {
			if (walls.tile[tile1].mapSwitch) {
				System.out.println("teleportation :" + walls.tile[tile1].mapSwitchNumber);
				Integer[] coord = Map.getCoordonneesTeleportations().get(walls.tile[tile1].mapSwitchNumber);
				gp.setCurrentMap(coord[0]);
				entite.setCoordonnees(coord[1],coord[2]);
				entite.setNumMap(coord[0]);
			}
		}
	}

	public void checkCollision(EntiteMobile entite) {
		
		int topOffset = GamePanel.tileSize / 3;
		int sideOffset = GamePanel.tileSize / 4;

		int LeftColumn = (entite.x + sideOffset) / GamePanel.tileSize;
		int RightColumn = (entite.x - sideOffset + GamePanel.tileSize) / GamePanel.tileSize;
		int UpperRow = (entite.y + topOffset) / GamePanel.tileSize;
		int BotRow = (entite.y + GamePanel.tileSize) / GamePanel.tileSize;

		int tile1, tile2;
		
		checkCollisionEntite(entite, entite.orientation, entite.x, entite.y);
		
		Walls walls = gp.getCurrentWalls();
		

		if (entite.collisionON != true) {
			
		switch (entite.orientation) {
		case "nord":
			UpperRow = (entite.y + topOffset - entite.speed) / GamePanel.tileSize;
			if (UpperRow < 0 || BotRow >= GamePanel.maxScreenRow || RightColumn >= GamePanel.maxScreenColumn
					|| LeftColumn < 0) {
				entite.collisionON = true;
			} else {
				tile1 = walls.matrix[UpperRow][LeftColumn];
				tile2 = walls.matrix[UpperRow][RightColumn];

				if (walls.tile[tile1].collision || walls.tile[tile2].collision) {
					entite.collisionON = true;
				}
				
			}
			break;
		case "sud":
			BotRow = (entite.y + GamePanel.tileSize + entite.speed) / GamePanel.tileSize;
			if (UpperRow < 0 || BotRow >= GamePanel.maxScreenRow || RightColumn >= GamePanel.maxScreenColumn
					|| LeftColumn < 0) {
				entite.collisionON = true;
			} else {
				tile1 = walls.matrix[BotRow][LeftColumn];
				tile2 = walls.matrix[BotRow][RightColumn];

				if (walls.tile[tile1].collision || walls.tile[tile2].collision) {
					entite.collisionON = true;
				}
			
			}
			break;
		case "est":
			RightColumn = (entite.x - sideOffset + GamePanel.tileSize + entite.speed) / GamePanel.tileSize;
			if (UpperRow < 0 || BotRow >= GamePanel.maxScreenRow || RightColumn >= GamePanel.maxScreenColumn
					|| LeftColumn < 0) {
				entite.collisionON = true;
			} else {
				tile1 = walls.matrix[UpperRow][RightColumn];
				tile2 = walls.matrix[BotRow][RightColumn];

				if (walls.tile[tile1].collision || walls.tile[tile2].collision) {
					entite.collisionON = true;
				}
				
			}
			break;
		case "ouest":
			LeftColumn = (entite.x + sideOffset - entite.speed) / GamePanel.tileSize;
			if (UpperRow < 0 || BotRow >= GamePanel.maxScreenRow || RightColumn >= GamePanel.maxScreenColumn
					|| LeftColumn < 0) {
				entite.collisionON = true;
			} else {
				tile1 = walls.matrix[UpperRow][LeftColumn];
				tile2 = walls.matrix[BotRow][LeftColumn];

				if (walls.tile[tile1].collision || walls.tile[tile2].collision) {
					entite.collisionON = true;
				}
				
				
			}

			break;
		default:
			System.out.println("c'est cassé");
		}}
	}
	
	private void checkCollisionEntite(Entite entiteTest, String direction, int xEntite, int yEntite) {
		
		List<Entite> entitelist = gp.getListeEntites(0);
		
		for (int k = 0; k < entitelist.size(); k++) {
			
			Entite entite = entitelist.get(k);
			
			if(entite != null && this.listeCollider.contains(entite.nom)) { 
			
			int x = (entite.getX() + GamePanel.tileSize/2)/GamePanel.tileSize;
			int y = (entite.getY() + GamePanel.tileSize/2)/GamePanel.tileSize;
			
			
			
			switch (direction) {
			case "nord":
				if (yEntite/GamePanel.tileSize == y && xEntite/GamePanel.tileSize == x && !entite.traversable) {
					entiteTest.collisionON = true;
				}
				break;
			case "sud":
				if (yEntite/GamePanel.tileSize == y - 1 && xEntite/GamePanel.tileSize == x && !entite.traversable) {
					entiteTest.collisionON = true;
				}
				break;
			case "est":
				if (xEntite/GamePanel.tileSize + 1 == x && yEntite/GamePanel.tileSize == y - 1 && !entite.traversable) {
					entiteTest.collisionON = true;
				}
				break;
			case "ouest":
				if (xEntite/GamePanel.tileSize == x && yEntite/GamePanel.tileSize == y - 1 && !entite.traversable) {
					entiteTest.collisionON = true;
				}
				break;
			default:
				System.out.println("c'est cassé");
				break;
			}
			if (entite.collisionON) {
				System.out.println("j'ai détecté une entité attention à vous");
			}
			
		}
		}

	}
}
```