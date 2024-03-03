---
title: "UART"
permalink: /UART/
author_profile: false
redirect_from:
  - /UART
---

Download an archive of the project : [Click here](/files/UART.zip)

Small implementation of an UART communication between a computer & FPGA or two FPGA 

Transmission Unit :
==

```vhdl
library IEEE;
use IEEE.std_logic_1164.all;

entity TxUnit is
  port (
    clk, reset : in std_logic;
    enable     : in std_logic;
    ld         : in std_logic;
    txd        : out std_logic;
    regE       : out std_logic;
    bufE       : out std_logic;
    data       : in std_logic_vector(7 downto 0));
end TxUnit;

architecture behavorial of TxUnit is

	type t_etat is (idle, chargementRegistre, debutEmission, emission, finEmission);
	signal etat : t_etat;

begin
	
	process(clk, reset)
		variable cpt_bit : natural; -- compte le nombre d'octets envoyés
		variable BufferT : std_logic_vector(7 downto 0); -- buffer tampon
		variable RegisterT : std_logic_vector(7 downto 0); -- registre tampon
		variable bufI : std_logic; -- statut de bufE
	begin
		if(reset = '0') then
			-- reset du compteur de bit
			cpt_bit := 7;
			-- reset des tampons
			BufferT := (others => 'U');
			RegisterT := (others => 'U');
			-- reset de l'état du buffer
			bufI := '1';
			
			-- reset des sorties
			txd <= '1';
			regE <= '1';
			bufE <= '1';
			
			-- état de base
			etat <= idle;
			
		elsif(rising_edge(clk)) then
			-- front montant de l'horloge
			case etat is
				when idle =>
					-- état d'attente d'un ordre d'envoi
					if(ld = '1') then
						cpt_bit := 7;
						bufE <= '0';
						regE <= '1';
						bufI := '0';
						BufferT := data;
						etat <= chargementRegistre;
					else
						-- aucun ordre : on ne fait rien
						null;
					end if;
				
				when chargementRegistre =>
					-- on charge le contenu du buffer tampon dans le registre tampon
					RegisterT := BufferT;
					bufE <= '1'; -- tampon est plein
					bufI := '1';
					regE <= '0';
					etat <= debutEmission;
					
				when debutEmission =>
					-- on stocke une potentielle deuxième donnée à envoyer après la transmission en cours
					if (ld = '1' and bufI = '1') then
						bufferT := data;
						bufE <= '0';
						bufI := '0';
					end if;
					
					
					-- on lance l'émission par l'envoi du bit de start txd=0
					if (enable = '1') then
						txd <= '0';
						cpt_bit := 7;
						etat <= emission;
					else
						null;
					end if;
				
				when emission => 
					-- on stocke une potentielle deuxième donnée à envoyer après la transmission en cours
					if (ld = '1' and bufI = '1') then
						bufferT := data;
						bufE <= '0';
						bufI := '0';
					end if;
					
					
					-- on émet les 8 bits contenus dans registreT sur txd
					if (enable = '1' and cpt_bit > 0) then
						txd <= registerT(cpt_bit);
						cpt_bit := cpt_bit - 1;
					elsif (enable = '1' and cpt_bit = 0) then
						txd <= RegisterT(0);
						etat <= finEmission;
					end if;
					
				when finEmission =>
					-- on stocke une potentielle deuxième donnée à envoyer après la transmission en cours
					if (ld = '1' and bufI = '1') then
						bufferT := data;
						bufE <= '0';
						bufI := '0';
					end if;
					
					
					-- on termine l'émession par le bit de stop txd = 1
					if (enable = '1' and bufI = '1') then
						txd <= '1';
						regE <= '1'; -- on est plus busy sur le registre
						etat <= idle;
					elsif (enable = '1' and bufI = '0') then
						txd <= '1';
						regE <= '1'; -- on est plus busy sur le registre
						etat <= chargementRegistre;
					end if;
					
			end case;
		end if;
	end process;
	
end behavorial;
```

Reception Unit :
==

```vhdl
library IEEE;
use IEEE.std_logic_1164.all;

entity RxUnit is
  port (
    clk, reset       : in  std_logic;
    enable           : in  std_logic;
    read             : in  std_logic;
    rxd              : in  std_logic;
    data             : out std_logic_vector(7 downto 0);
    Ferr, OErr, DRdy : out std_logic);
end RxUnit;

architecture behavorial of RxUnit is

		COMPONENT ctrlReception
	PORT(
		reset : IN std_logic;
		tmpclk : IN std_logic;
		tmprxd : IN std_logic;
		read : IN std_logic;
		clk : IN std_logic;          
		FErr : OUT std_logic;
		OErr : OUT std_logic;
		DRdy : OUT std_logic;
		data : OUT std_logic_vector(7 downto 0)
		);
	END COMPONENT;
	
	COMPONENT compteur16
	PORT(
		enable : IN std_logic;
		reset : IN std_logic;
		rxd : IN std_logic;          
		tmpclk : OUT std_logic;
		tmprxd : OUT std_logic
		);
	END COMPONENT;
	
	signal tmpclk_int : std_logic;
	signal tmprxd_int : std_logic;

begin
	
	-- DRdy=1 : donnée reçue
	-- Ferr=1 : trame reçue erronée
	-- OErr=1 : une donnée est prête mais pas traitée à temps par le processeur
	
	Inst_ctrlReception: ctrlReception PORT MAP(
		reset => reset,
		tmpclk => tmpclk_int,
		tmprxd => tmprxd_int,
		read => read,
		clk => clk,
		FErr => FErr,
		OErr => OErr,
		DRdy => DRdy,
		data => data 
	);
	
	Inst_compteur16: compteur16 PORT MAP(
		enable => enable,
		reset => reset,
		rxd => rxd,
		tmpclk => tmpclk_int,
		tmprxd => tmprxd_int
	);

end behavorial;
```