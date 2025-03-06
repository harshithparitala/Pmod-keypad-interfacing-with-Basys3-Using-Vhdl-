# Pmod-keypad-interfacing-with-Basys3-UsingVhdl


Pmod_Keypad module 
```

library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity pmod_keypad is
    Port (
        clk : in STD_LOGIC;
        row : in STD_LOGIC_VECTOR(3 downto 0);
        col : out STD_LOGIC_VECTOR(3 downto 0);
        key : out STD_LOGIC_VECTOR(3 downto 0);
        key_valid : out STD_LOGIC  -- 1 when a key is pressed
    );
end pmod_keypad;

architecture Behavioral of pmod_keypad is
    constant BITS         : integer := 20;
    constant ONE_MS_TICKS : integer := 100_000;  -- 1ms @ 100MHz
    constant SETTLE_TIME  : integer := 100;      -- 1us @ 100MHz

    signal key_counter : unsigned(BITS-1 downto 0) := (others => '0');
    signal rst         : STD_LOGIC := '0';
    signal key_detected : STD_LOGIC := '0';

begin
    -- Counter Process
    process(clk)
    begin
        if rising_edge(clk) then
            if rst = '1' then
                key_counter <= (others => '0');
            else
                key_counter <= key_counter + 1;
            end if;
        end if;
    end process;

    -- Keypad Scanning Process
    process(clk)
    begin
        if rising_edge(clk) then
            rst <= '0';  -- Default
            key_valid <= '0'; -- Default no key press

            case to_integer(key_counter) is
                when ONE_MS_TICKS =>
                    col <= "0111";
                when ONE_MS_TICKS + SETTLE_TIME =>
                    case row is
                        when "0111" => key <= "0001"; key_valid <= '1'; -- 1
                        when "1011" => key <= "0100"; key_valid <= '1'; -- 4
                        when "1101" => key <= "0111"; key_valid <= '1'; -- 7
                        when "1110" => key <= "0000"; key_valid <= '1'; -- 0
                        when others => key_valid <= '0'; 
                    end case;

                when 2 * ONE_MS_TICKS =>
                    col <= "1011";
                when 2 * ONE_MS_TICKS + SETTLE_TIME =>
                    case row is
                        when "0111" => key <= "0010"; key_valid <= '1'; -- 2
                        when "1011" => key <= "0101"; key_valid <= '1'; -- 5
                        when "1101" => key <= "1000"; key_valid <= '1'; -- 8
                        when "1110" => key <= "1111"; key_valid <= '1'; -- F (Unused key)
                        when others => key_valid <= '0'; 
                    end case;

                when 3 * ONE_MS_TICKS =>
                    col <= "1101";
                when 3 * ONE_MS_TICKS + SETTLE_TIME =>
                    case row is
                        when "0111" => key <= "0011"; key_valid <= '1'; -- 3
                        when "1011" => key <= "0110"; key_valid <= '1'; -- 6
                        when "1101" => key <= "1001"; key_valid <= '1'; -- 9
                        when "1110" => key <= "1110"; key_valid <= '1'; -- E (Unused key)
                        when others => key_valid <= '0'; 
                    end case;

                when 4 * ONE_MS_TICKS =>
                    col <= "1110";
                when 4 * ONE_MS_TICKS + SETTLE_TIME =>
                    case row is
                        when "0111" => key <= "1010"; key_valid <= '1'; -- A
                        when "1011" => key <= "1011"; key_valid <= '1'; -- B
                        when "1101" => key <= "1100"; key_valid <= '1'; -- C
                        when "1110" => key <= "1101"; key_valid <= '1'; -- D
                        when others => key_valid <= '0'; 
                    end case;

                when 5 * ONE_MS_TICKS =>
                    rst <= '1';  -- Reset counter after full scan

                when others => null;
            end case;
        end if;
    end process;
end Behavioral;

`````

Seven Segment module "
````
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;

entity sseg_display is
    Port (
        hex : in STD_LOGIC_VECTOR(3 downto 0);
        seg : out STD_LOGIC_VECTOR(6 downto 0)
    );
end sseg_display;

architecture Behavioral of sseg_display is
begin
    process(hex)
    begin
        case hex is
            when "0000" => seg <= "1000000"; -- '0'
            when "0001" => seg <= "1111001"; -- '1'
            when "0010" => seg <= "0100100"; -- '2'
            when "0011" => seg <= "0110000"; -- '3'
            when "0100" => seg <= "0011001"; -- '4'
            when "0101" => seg <= "0010010"; -- '5'
            when "0110" => seg <= "0000010"; -- '6'
            when "0111" => seg <= "1111000"; -- '7'
            when "1000" => seg <= "0000000"; -- '8'
            when "1001" => seg <= "0010000"; -- '9'
            when "1010" => seg <= "0001000"; -- 'A'
            when "1011" => seg <= "0000011"; -- 'B'
            when "1100" => seg <= "1000110"; -- 'C'
            when "1101" => seg <= "0100001"; -- 'D'
            when "1110" => seg <= "0000110"; -- 'E'
            when "1111" => seg <= "0001110"; -- 'F'
            when others => seg <= "1111111"; -- Blank display
        end case;
    end process;
end Behavioral;

````
counter-n module 
`````
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity counter_n is
    generic(
        BITS : integer := 4
    );
    port(
        clk  : in  STD_LOGIC;
        rst  : in  STD_LOGIC;
        tick : out STD_LOGIC;
        q    : out STD_LOGIC_VECTOR(BITS-1 downto 0)
    );
end counter_n;

architecture Behavioral of counter_n is
    signal rCounter : unsigned(BITS-1 downto 0) := (others => '0');
begin

    -- Asynchronous reset with clock-synchronized increment
    process(clk, rst)
    begin
        if rst = '1' then          -- Async reset (priority)
            rCounter <= (others => '0');
        elsif rising_edge(clk) then -- Clocked increment
            rCounter <= rCounter + 1;
        end if;
    end process;

    -- Direct output connection
    q <= std_logic_vector(rCounter);

    -- Terminal count detection
    tick <= '1' when rCounter = to_unsigned(2**BITS - 1, BITS) 
           else '0';

end Behavioral;
``````

Keypad_top module 
````
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;

entity keypad_top is
    Port (
        clk  : in  STD_LOGIC;
        JA   : inout STD_LOGIC_VECTOR(7 downto 0);
        seg  : out STD_LOGIC_VECTOR(6 downto 0);
        an   : out STD_LOGIC_VECTOR(3 downto 0);
        led  : out STD_LOGIC  -- Debugging LED for key_valid
    );
end keypad_top;

architecture Structural of keypad_top is
    signal key : STD_LOGIC_VECTOR(3 downto 0);
    signal key_valid : STD_LOGIC;

    component pmod_keypad
        Port (
            clk : in  STD_LOGIC;
            row : in  STD_LOGIC_VECTOR(3 downto 0);
            col : out STD_LOGIC_VECTOR(3 downto 0);
            key : out STD_LOGIC_VECTOR(3 downto 0);
            key_valid : out STD_LOGIC
        );
    end component;

    component sseg_display
        Port (
            hex : in  STD_LOGIC_VECTOR(3 downto 0);
            seg : out STD_LOGIC_VECTOR(6 downto 0)
        );
    end component;

begin
    -- Debugging LED: Turns ON when a key is detected
    led <= key_valid;

    -- Force anode always ON for debugging
    an <= "1110" when key_valid = '1' else "1110";  -- TEMPORARY FIX to see if display is working

    -- Keypad module
    KEYPAD_INST : pmod_keypad
        port map (
            clk => clk,
            row => JA(7 downto 4),
            col => JA(3 downto 0),
            key => key,
            key_valid => key_valid
        );

    -- Seven-segment display
    DISPLAY_INST : sseg_display
        port map (
            hex => key,
            seg => seg
        );

end Structural;

`````

xdc file 

````
## This file is a general .xdc for the Basys3 rev B board
## To use it in a project:
## - uncomment the lines corresponding to used pins
## - rename the used ports (in each line, after get_ports) according to the top level signal names in the project
 
## Clock signal
set_property PACKAGE_PIN W5 [get_ports clk]
    set_property IOSTANDARD LVCMOS33 [get_ports clk]
    create_clock -add -name sys_clk_pin -period 10.00 -waveform {0 5} [get_ports clk]
 
##7 segment display
set_property PACKAGE_PIN W7 [get_ports {seg[0]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[0]}]
set_property PACKAGE_PIN W6 [get_ports {seg[1]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[1]}]
set_property PACKAGE_PIN U8 [get_ports {seg[2]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[2]}]
set_property PACKAGE_PIN V8 [get_ports {seg[3]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[3]}]
set_property PACKAGE_PIN U5 [get_ports {seg[4]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[4]}]
set_property PACKAGE_PIN V5 [get_ports {seg[5]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[5]}]
set_property PACKAGE_PIN U7 [get_ports {seg[6]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {seg[6]}]
 
set_property PACKAGE_PIN U2 [get_ports {an[0]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {an[0]}]
set_property PACKAGE_PIN U4 [get_ports {an[1]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {an[1]}]
set_property PACKAGE_PIN V4 [get_ports {an[2]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {an[2]}]
set_property PACKAGE_PIN W4 [get_ports {an[3]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {an[3]}]
 
##Pmod Header JA
##Sch name = JA1
set_property PACKAGE_PIN J1 [get_ports {JA[0]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[0]}]
#Sch name = JA2
set_property PACKAGE_PIN L2 [get_ports {JA[1]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[1]}]
#Sch name = JA3
set_property PACKAGE_PIN J2 [get_ports {JA[2]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[2]}]
#Sch name = JA4
set_property PACKAGE_PIN G2 [get_ports {JA[3]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[3]}]
#Sch name = JA7
set_property PACKAGE_PIN H1 [get_ports {JA[4]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[4]}]
#Sch name = JA8
set_property PACKAGE_PIN K2 [get_ports {JA[5]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[5]}]
#Sch name = JA9
set_property PACKAGE_PIN H2 [get_ports {JA[6]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[6]}]
#Sch name = JA10
set_property PACKAGE_PIN G3 [get_ports {JA[7]}]
    set_property IOSTANDARD LVCMOS33 [get_ports {JA[7]}]
 
set_property PACKAGE_PIN U16 [get_ports {debug_leds[0]}]   # LD0
set_property PACKAGE_PIN E19 [get_ports {debug_leds[1]}]   # LD1
set_property PACKAGE_PIN U19 [get_ports {debug_leds[2]}]   # LD2
set_property PACKAGE_PIN V19 [get_ports {debug_leds[3]}]   # LD3
set_property PACKAGE_PIN W18 [get_ports {debug_leds[4]}]   # LD4
set_property PACKAGE_PIN U15 [get_ports {debug_leds[5]}]   # LD5
set_property PACKAGE_PIN U14 [get_ports {debug_leds[6]}]   # LD6
set_property PACKAGE_PIN V14 [get_ports {debug_leds[7]}]   # LD7


set_property PACKAGE_PIN U16 [get_ports led]
set_property IOSTANDARD LVCMOS33 [get_ports led]

## Configuration options, can be used for all designs
set_property CONFIG_VOLTAGE 3.3 [current_design]
set_property CFGBVS VCCO [current_design]
`````
