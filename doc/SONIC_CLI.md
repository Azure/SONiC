Requirements
============
(1) Tacacs+ authorization.

(2) Like traditional CLI to support:

  Command completion by TAB, ? Help.

  Configure mode, Interface mode, BGP mode.

(3) No conflict with current SONIC commands and easy to reuse.

Major design
============

(1) SONIC CLI is running on host environment.

(2) Wrapping on current CLI to reuse it.

(3) SONIC CLI is developing based on VTYSH and support all commands of
    Quagga or FRR.

(4) Authorization by remote Tacacs+ server.


                +--------------------+
                |  Tacacs+ server    |
                +---+-----+-----+----+
                          | Authorization
     +--------------------v---------------------+
     | Host               |                     |
     |    +--------------------------------+    |     
     |    |        SONIC CLI               |    |      
     |    +--------------------------|------+   |      
     |     |             |           |          |
     |     |      +------------+     |          |
     |     |      |  Linux CLI |     |          |
     |     |      +------------+     |          |
     |     |             |           |          |
     +-----|---------+---------------|----------+                  |
           |             |           |              
           |   +---------+-------------+
     socket|   |        ConifgDB       |
           |   +-----------------------+
           |       |               |
    +-----------------+    +-----------------+                
    | BGP Docker      |    | App Docker      |                  
    |                 |    |                 |
    |  VTYSH Server   |    |   Enforcer      |
    |                 |    |                 |
    +-----------------+    +-----------------+                
     



How to reuse current CLI?
-------------------------

VTYSH has a good defined framework and easily to create a new command.
SONIC CLI just call linux command by a pipe.

Example:
1. call existing SONIC comand

DEFUN (show_interface_counters, show_interface_counters_cmd,
            "show interface counters",
            SHOW_STR
            INTERFACE_STR
            "interfaces counters\n")
{
    execute_linux_cmd_pipe("show interface counters -a");
    return CMD_SUCCESS;
}

2. call linux command

DEFUN (show_timezone,
            show_timezone_cmd,
            "show timezone",
            SHOW_STR
            "timezone\n")
{
    execute_linux_cmd_pipe("timedatectl status");
    return CMD_SUCCESS;
}

3. call existing SONIC comand to config

DEFUN (cli_aaa_auth_login_one,
            aaa_auth_login_one_cmd,
            "aaa authentication login (local|tacacs+)",
            AAA_HELP_STR
            AAA_AUTHENTICATION_HELP_STR
            AAA_LOGIN_HELP_STR
            "Local authentication\n"
            "TACACS+ authentication\n")
{
    EXEC_CMD_WITH_PARAM("sudo config aaa authentication login %s", argv[0]);

    if(author_update_config())
        vty_out(vty, "Update TACACS+ configuration fail\n");

    return CMD_SUCCESS;
}



Support all commands of Quagga/FRR
----------------------------------

1.  Compiling Quagga to generate a file named vtysh_cmd.c under src/sonic-quagga/vtysh/

2.  Copy it to src/sonic-cli/vtysh/

3. compile sonic-cli

4.  Get all commands of Quagga



What CLI actually has been done with Quagga?

1.  CLI use vtysh_cmd.c to get the command tree from Quagga.

DEFSH (VTYSH_ZEBRA, show_ip_route_prefix_cmd_vtysh, 
        "show ip route A.B.C.D/M\",
        "Show running system information\n"
        "IP information\n"
        "IP routing table\n"
        "IP prefix <network>/<length>, e.g., 35.0.0.0/8\n")
2.  CLI connects to vtysh server of zebra and bgpd.

3.  CLI send command to zebra. zebra will execute the command and return
    result to CLI.

4.  CLI printout the result.

Different between SONIC CLI and VTYSH
-------------------------------------

1.  SONIC CLI is an independent repository and can work without
    Quagga/FRR.

2.  SONIC CLI is running in host linux and easily to call existing
    python CLI and Linux commands.

3.  SONIC CLI supports authentication and authorization.

Authorization
-------------

1.  User enters to SONIC CLI by default when logins to switch.

  /etc/passwd
  yongcan.wyc:x:1003:1000:remote\_user:/home/yongcan.wyc:/usr/bin/cli

2.  Get privilege level after do authentication by Tacacs+ server.

3.  Create a command tree based on privilege level.

Admin user gets a tree of all commands.

  Other user gets a tree without high risk commands. For example, there
  is no command to enter configure mode, also no command to enter to
  Linux.

4.  Before executing any command, SONIC CLI will send it to Tacacs+
    server to see if user has authorization to do it.

5.  All users including no admin user join to the sudo group and docker
    group, as some show commonds need sudo or docker. So if user passes
    authorization by Tacacs+, it can show anything successfully.
