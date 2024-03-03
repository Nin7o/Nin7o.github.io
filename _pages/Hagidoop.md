---
title: "Hagidoop"
permalink: /Hagidoop/
author_profile: false
redirect_from:
  - /Hagidoop
---

Download an archive of the project : [Click here](/files/hagidoop.zip)

I've worked on the majority of this project. It sends and receive files via sockets, and start map/reduce operations with RMI.

Object implementing a multi-input single-output communication via sockets of KV (key-value) objects (to gather the results of the computations): 

NetworkReaderWriter
==

```java
package io;
import java.io.*;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.ReentrantLock;

import interfaces.KV;
import interfaces.NetworkReaderWriter;

/**
 * Implémente le lecteur/rédacteur sur le réseau.
 * Il permet de lire/écrire sur le lien entre les résultats du Map et l'entrée
 * du reduce.
 */
public class ImplNetworkReaderWriter implements NetworkReaderWriter, Cloneable {

    public static String ANSI_RESET = "\u001B[0m";
    public static String ANSI_RED = "\u001B[31m";
	public static String ANSI_GREEN = "\u001B[32m";
    public static String ANSI_YELLOW = "\u001B[33m";
    public static String ANSI_BLUE = "\u001B[34m";

    private Socket connectedSocket;
    private ServerSocket serverSocket;

    private List<Socket> serverConnectedSockets = new ArrayList<>();
    private List<InputStream> inputStreams = new ArrayList<>();
    private List<ObjectInputStream> objectInputStreams = new ArrayList<>();

    private OutputStream outputStream;
    private ObjectOutputStream objectOutputStream;

    private String address;
    private int port;

    private final ReentrantLock rlock = new ReentrantLock();
    private final ReentrantLock wlock = new ReentrantLock();

    public ImplNetworkReaderWriter(String address, int port, Socket connectedSocket) {
        this.address = address;
        this.port = port;
        this.connectedSocket = connectedSocket;

    }

    private int l = 0;

    
    /** 
     * Permet de lire un KV sur le réseau
     * @return renvoie le KV lu
     */
    @Override
    public KV read() {
        //Read ça veut dire qu'on read depuis un des workers et qu'on est le server
        // on va donc lire depuis un des inputStreams
        // si la lecture est null, on ferme le socket et on le retire de la liste

        //si la liste est vide, on return null

        //sinon on lit depuis le premier de la liste

        //si la lecture est null, on ferme le socket et on le retire de la liste

        //sinon on return la lecture
        rlock.lock();
        try {

            l = objectInputStreams.size();
            if (l == 0) {
                return null;
            }    

            KV kv = null;
            for (int i = 0; i < l; i++) {
                try {
                    kv = (KV) objectInputStreams.get(i).readObject();
                    if (kv == null) {
                        serverConnectedSockets.get(i).close();
                        serverConnectedSockets.remove(i);
                        inputStreams.get(i).close();
                        inputStreams.remove(i);
                        objectInputStreams.get(i).close();
                        objectInputStreams.remove(i);
                        System.out.println(ANSI_BLUE + "[-] Disconnection from a client" + ANSI_RESET);
                        l--;
                        i--;
                    } else {
                        return kv;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

            return null;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        } finally {
            rlock.unlock();
        }
    }

    
    /** 
     * Permet d'écrire un KV sur le réseau
     * @param record le KV à écrire
     */
    @Override
    public void write(KV record) {
        wlock.lock();
        try {
            objectOutputStream.writeUnshared(record);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            wlock.unlock();
        }

    }

    /**
     * Ouvre le lecteur/rédacteur en tant que serveur
     */
    @Override
    public void openServer() {
        try {
            System.out.println(ANSI_GREEN + "[+] Opening NetworkReaderWriter server on : " + address + ":" + port + ANSI_RESET);
            serverSocket = new ServerSocket(port);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * Ouvre le lecteur/rédacteur en tant que client
     * Essaye alors de se connecter au serveur
     */
    @Override
    public void openClient() {
        try {
            // create the client socket connected to the server
            connectedSocket = new Socket(address, port);
            System.out.println(ANSI_GREEN + "[+] Connected NetworkReaderWriter to the server on : " + address + ":" + port + ANSI_RESET);

            //get the output stream from the connected socket
            outputStream = connectedSocket.getOutputStream();
            objectOutputStream = new ObjectOutputStream(outputStream);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    
    /** 
     * Accepte une connection sur un socket
     * @return renvoie une copie de l'objet, ou le socket à été accepté ett stocké
     */
    @Override
    public NetworkReaderWriter accept() {
        // accept a connection on the server socket
        // here we don't care about the return value, we just want to add the socket to the list
        // once all the sockets are added to the list, we can start reading from them
        // so we return a clone of this object
        try {
            Socket s = serverSocket.accept();
            serverConnectedSockets.add(s);
            
            //get the input stream from the connected socket
            InputStream is = s.getInputStream();
            inputStreams.add(is);
            ObjectInputStream ois = new ObjectInputStream(is);
            objectInputStreams.add(ois);
            
            System.out.println(ANSI_BLUE + "[+] New connection to the server !" + ANSI_RESET);

            return (NetworkReaderWriter) super.clone();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * Ferme le lecteur/rédacteur en tant que serveur
     */
    @Override
    public void closeServer() {
        if (serverSocket != null) {
            try {
                serverSocket.close();
                System.out.println(ANSI_RED + "[-] Closing NetworkReaderWriter server" + ANSI_RESET);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * Ferme le lecteur/rédacteur en tant que client
     */
    @Override
    public void closeClient() {
        if (connectedSocket != null) {
            try {
                objectOutputStream.writeObject(null);
                connectedSocket.close();
                System.out.println(ANSI_RED + "[-] Disconnected from NetworkReaderWriter server " + ANSI_RESET);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    
}
```

File transfer server :
==

```java
package hdfs;

import java.net.*;
import java.util.Scanner;

import config.Project;

import java.io.*;

/**
 * Côté serveur de hdfs : reçoit des fichiers du client et répond à différentes 
 * requêtes.
 */
public class HdfsServer {

    public static String ANSI_RESET = "\u001B[0m";
    public static String ANSI_RED = "\u001B[31m";
	public static String ANSI_GREEN = "\u001B[32m";
    public static String ANSI_YELLOW = "\u001B[33m";
    public static String ANSI_BLUE = "\u001B[34m";

    private static int port = 0;
    private static boolean actionEnded = false;
    private static final int MAX_ITER = 1000000;
    
    
    /** 
     * Permet de lancer un serveur HDFS
     * @param args le port sur lequel le serveur va écouter
     */
    public static void main(String[] args) {

        // check usage
        if (args.length != 1) {
            System.out.println("Usage: java HdfsServer <port>");
            System.exit(1);
        }

        port = Integer.parseInt(args[0]);

        // set and reset environment
        createDir();
        //deleteDirContent(); // a enlever à un moment pour une gestion plus propre

        try {

            // create the server socket
            ServerSocket serverSocket;
            serverSocket = new ServerSocket(port);
            System.out.println(ANSI_BLUE + "Hagidoop Distributed File System started on port " + port + ANSI_RESET);

            //to remove warnings
            //have to be changed so that an order can close the server
            while (port < MAX_ITER) {
                
                //accept the client connection
                Socket socket = serverSocket.accept();
                System.out.println(ANSI_GREEN + "[+] Connection d'un client" + ANSI_RESET);

                //get the input stream from the connected socket
                InputStream inputStream = socket.getInputStream();
                ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);

                //get the output stream from the connected socket
                OutputStream outputStream = socket.getOutputStream();
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
                    
                //receive all the chunks / order from the client
                //will break when the order is completed
                while (true) {

                    try {
                        //receive the chunk
                        DataChunk chunk = (DataChunk) objectInputStream.readObject();
        
                        //break order
                        if (chunk.action == 4) {
                            System.out.println(ANSI_BLUE + "File " + chunk.name + " received" + ANSI_RESET);
                            System.out.println(ANSI_RED + "[-] Déconnexion du client\n" + ANSI_RESET);
                            break;
                        }

                        // else, do the action
                        switch (chunk.action) {

                            case 0:
                                System.out.println(ANSI_BLUE + "File " + chunk.name + " requested" + ANSI_RESET);
                                read(chunk.name, objectOutputStream);
                                actionEnded = true;
                                break;

                            case 1:
                                write(chunk);
                                break;

                            case 2:    
                                delete(chunk, objectOutputStream);
                                System.out.println(ANSI_BLUE + "File " + chunk.name + " deleted" + ANSI_RESET);
                                actionEnded = true;
                                break;

                            case 5:
                                System.out.println(ANSI_BLUE + "Ping received" + ANSI_RESET);
                                actionEnded = true;
                                break;

                            case 6:
                                System.out.println(ANSI_BLUE + "List files requested" + ANSI_RESET);
                                listFiles(objectOutputStream);
                                actionEnded = true;
                                break;
                        
                            default:
                                System.out.println(ANSI_YELLOW + "[!] Wrong action flag" + ANSI_RESET);
                                break;
                        }

                        System.gc();

                        //if the action server side is ended, break the loop
                        if (actionEnded) {
                            actionEnded = false;
                            System.out.println(ANSI_RED + "[-] Déconnexion du client\n" + ANSI_RESET);
                            break;
                        }

                    } catch (java.net.SocketException e) {
                        System.out.println(ANSI_RED + "[Error] Socket Exception : Back to waiting state of the server\nThe current action has stopped abruptly\nConsider deleting all generated files and trying again");
                        break;
                    }
                
                }

            }

            //close the socket
            serverSocket.close();

            System.out.println(ANSI_RED + "Server closed" + ANSI_RESET);
            
        } catch (Exception e) {
            e.printStackTrace();
        }      
        
    }

    
    /** 
     * Renvoie la liste de ses fichiers au client
     * @param oos l'ObjectOutputStream sur lequel écrire la liste
     */
    private static void listFiles(ObjectOutputStream oos) {
        try {

            String folderPath = Project.NODEPATH + port;
            File folder = new File(folderPath);

            String files = "";

            for (File file : folder.listFiles()) {
                files += file.getName() + "\n";
            }

            oos.writeUnshared(new DataChunk(files, "", 6));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * Crée le dossier de travail du noeud
     */
    private static void createDir() {
        try {
            String folderPath = Project.NODEPATH + port;
            File folder = new File(folderPath);
            folder.mkdir();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * Supprime le contenu du dossier de travail du noeud
     */
    @SuppressWarnings("unused")
    private static void deleteDirContent() {
        try {
            String folderPath = Project.NODEPATH + port;
            File folder = new File(folderPath);
            if (folder.exists()) {
                for (File file : folder.listFiles()) {
                    file.delete();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    
    /** 
     * Lit en fichier et renvoie sont contenu au client
     * @param filename le nom du fichier
     * @param objectOutputStream l'ObjectOutputStream sur lequel écrire le fichier
     */
    private static void read(String filename, ObjectOutputStream objectOutputStream) {
        try {

            String absolutePath = Project.NODEPATH + port + "/" + port + "-" + filename;
            File file = new File(absolutePath);

            if (!file.exists()) {
                absolutePath = Project.NODEPATH + port + "/" + port + "-kv-" + filename;
            }

            // instanciate the file reader
            FileReader fr = new FileReader(absolutePath);
            Scanner scanner = new Scanner(fr);

            // set the delimiter to \n
            String delimiterRegex = "(?<=\\n)";
			scanner.useDelimiter(delimiterRegex);

            // get the max size of a dataChunk
            long sizeMax = DataChunk.sizeMax;

            // stringbuilder to store the chunk
            StringBuilder sb = new StringBuilder();
            long size = 0;

            DataChunk chunk = new DataChunk("", filename, 1);

            String word = "";

            // read the file
            while (scanner.hasNext()) {

                word = scanner.next();
                sb.append(word);

                int wordSize = word.getBytes("UTF-8").length;
                size += wordSize;

                // if the size of the chunk is bigger than the max size, send it
                if (size >= sizeMax) {
                    chunk.text = sb.toString();
                    objectOutputStream.writeUnshared(chunk);
                    objectOutputStream.flush();
                    objectOutputStream.reset();
                    sb = new StringBuilder();
                    size = 0;
                }

            }

            // if there is still data in the stringbuilder, send it
            if (size > 0) {
                objectOutputStream.writeUnshared(new DataChunk(sb.toString(), filename, 1));
                objectOutputStream.flush();
                objectOutputStream.reset();
            }

            // send a null chunk to indicate the end of the reading
            objectOutputStream.writeUnshared(new DataChunk("", filename, 4));
            objectOutputStream.flush();
            objectOutputStream.reset();

            System.out.println(ANSI_GREEN + "File " + filename + " sent" + ANSI_RESET);

            scanner.close();

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    
    /** 
     * Écrit un nouveau fichier
     * @param chunk le chunk de données à écrire - contient le nom du fichier et son contenu
     */
    private static void write(DataChunk chunk) {
        try {

            String absolutePath = Project.NODEPATH + port + "/" + chunk.name;

            FileWriter fw = new FileWriter(absolutePath, true);
            BufferedWriter bw = new BufferedWriter(fw);

            // write the chunk in the file
            bw.write(chunk.getText());

            bw.close();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    
    /** 
     * Supprime un fichier
     * @param chunk le nom du fichier
     * @param objectOutputStream l'ObjectOutputStream sur lequel renvoyer l'effectivité de la supression
     */
    private static void delete(DataChunk chunk, ObjectOutputStream objectOutputStream) {
        try {

            // file to delete
            String absolutePath = Project.NODEPATH + port + "/" + port + "-" + chunk.name;
            File file = new File(absolutePath);

            if (!file.exists()) {
                absolutePath = Project.NODEPATH + port + "/" + port + "-kv-" + chunk.name;
            }

            file = new File(absolutePath);

            // delete the file
            file.delete();

            // send a null chunk to indicate the end of the deleting
            objectOutputStream.writeUnshared(new DataChunk("", "", 2));
            objectOutputStream.flush();
            objectOutputStream.reset();

        } catch (Exception e) {
            // error handling
            try {
            objectOutputStream.writeUnshared(new DataChunk("", "", 3));
            objectOutputStream.flush();
            objectOutputStream.reset();
            } catch (Exception e2) {
                e2.printStackTrace();
            }
        }
    }


}
```
