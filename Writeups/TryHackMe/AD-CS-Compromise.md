1. Resumen Ejecutivo Este informe detalla la intrusión y escalada de privilegios en el entorno de Active Directory za.tryhackme.loc. El ataque progresó desde una cuenta de usuario estándar hasta el compromiso total del Domain Controller (DC) mediante el abuso de servicios de certificados (AD CS) y técnicas de manipulación de tickets Kerberos.

2. Información del Entorno
  Dominio: za.tryhackme.loc
  Domain Controller (DC): THMDC (10.200.72.101)
  Punto de entrada: Usuario thomas.hunt
  Objetivo: Flag de Administrador en el DC.

3. Fase 1: Movimiento Lateral y GPO Abuse
  Se identificó que el usuario inicial pertenecía al grupo IT Support, el cual tenía permisos de administración local sobre servidores específicos gracias a una GPO mal configurada.
  Técnica: Modificación de Grupos Restringidos vía GPO.

  Resultado: Acceso administrativo a servidores de nivel intermedio.

4. Fase 2: Explotación de AD CS (ESC1)
  Se detectó una plantilla de certificado vulnerable en el servidor de certificados (CA). Esta plantilla permitía a usuarios del dominio solicitar certificados con Subject Alternative Name (SAN), lo que permite la suplantación de cualquier identidad.

  Extracción del Certificado
  Utilizando un certificado previamente obtenido (vulncert.pfx), se procedió a solicitar un Ticket Granting Ticket (TGT) para la cuenta Administrator.
  
5. Fase 3: Escalada con Rubeus y Solución de Errores
    El intento inicial de obtener el ticket falló debido a una desincronización horaria (Time Skew).

    Troubleshooting: Error c0000133
    El sistema devolvió el error STATUS_TIME_DIFFERENCE_AT_ISSUE.

    Causa: Diferencia de más de 5 minutos entre el cliente y el KDC.

    Solución: Sincronización manual con el DC:

    net time \\10.200.72.101 /set /y

    Generación del TGT (Pass-the-Certificate)
    Con la hora sincronizada, se generó un ticket:

    .\Rubeus.exe asktgt /user:Administrator /enctype:aes256 /certificate:vulncert.pfx /password:tryhackme /outfile:administrator.kirbi /domain:za.tryhackme.loc /dc:10.200.72.101  

6. Fase 4: Pass-the-Ticket con Mimikatz
    Para utilizar el ticket generado, se inyectó en el proceso lsass de la sesión actual.
    mimikatz # privilege::debug
    mimikatz # kerberos::ptt administrator.kirbi

7. Fase 5: Post-Explotación y Flag
    Con el ticket de Administrador de Dominio inyectado, se obtuvo acceso al sistema de archivos del DC de forma remota.

    dir \\THMDC.za.tryhackme.loc\c$\

    Localización de la Flag
    La flag final fue exfiltrada desde el escritorio del Administrador:

    type \\THMDC.za.tryhackme.loc\C$\Users\Administrator\Desktop\flag.txt


8. Recomendaciones de Mitigación (Remediación)
    Auditoría de AD CS: Revisar y deshabilitar la opción CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT en las plantillas de certificados de producción.

    Privileged Access Management: Limitar las cuentas que pueden editar GPOs críticas.

    Monitoreo: Implementar alertas para eventos de Kerberos inusuales (ej. AS-REQ con certificados o uso de tipos de cifrado débiles).





