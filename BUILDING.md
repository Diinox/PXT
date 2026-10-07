# Compilacion y ejecucion de PokeDash Pota

Se conserva C++11 y las mecanicas existentes. LuaJIT es la opcion predeterminada.
La alternativa USE_LUAJIT=OFF exige Lua 5.1. Los atributos de las pokeballs
dependen de loadstring: no seleccionar Lua 5.4 automaticamente.

## Windows x64

Requisitos: Visual Studio 2022 Build Tools con Desktop development with C++,
Windows SDK, CMake >= 3.21 y Git. Todas las bibliotecas deben ser x64 y
corresponder al compilador y configuracion.

Desde la raiz del proyecto, en PowerShell:

```powershell
git clone https://github.com/microsoft/vcpkg.git .tools/vcpkg
git -C .tools/vcpkg checkout 2976266a3b2aa358e0eed9e89bde2e0cf59daf0a
& .tools/vcpkg/bootstrap-vcpkg.bat -disableMetrics
$env:VCPKG_ROOT = (Resolve-Path .tools/vcpkg).Path
cmake --preset windows
cmake --build --preset windows --parallel 2
& ./build/windows/Release/tfs.exe
```

El manifiesto fija las dependencias al registro
f7423ee180c4b7f40d43402c2feb3859161ef625 (Boost 1.85.0, GMP 6.3.0,
PugiXML 1.14, MariaDB Connector/C 3.3.1 y LuaJIT revision 2023-01-04).
El manifiesto aplica la revision de empaquetado GMP 6.3.0#5 y fija sus
herramientas auxiliares para evitar el fallo de dumpbin con Autoconf 2.72.
CMake/vcpkg instala las bibliotecas
y copia sus DLL junto al ejecutable. No mezclar la antigua libmysql.dll
incluida con el nuevo conector MariaDB.

Para distribuir: copiar tfs.exe, sus DLL, config.lua y data/ a una carpeta
propia. Instalar Visual C++ Redistributable x64. No sobrescribir el ejecutable
historico antes de validar el nuevo.

## Linux x86-64: Ubuntu 24.04

```bash
sudo apt-get update
sudo apt-get install -y cmake ninja-build g++ pkg-config \
  libboost-system-dev libluajit-5.1-dev libmariadb-dev libgmp-dev libpugixml-dev
cmake --preset linux
cmake --build --preset linux --parallel 2
ldd build/linux/tfs
./build/linux/tfs
```

Compilar para la distribucion destino. No reutilizar src/tfs: esta ligado a
versiones antiguas de MySQL, Lua y Boost. Registrar las versiones exactas
de paquetes con dpkg-query -W: los repositorios reciben actualizaciones.

## Base de datos y directorio de trabajo

Usar una base de pruebas vacia y configurar sus credenciales en config.lua.
schema.sql incluye DROP TABLE, personajes y triggers con DEFINER root@localhost.
No importarlo sobre una base existente. El servidor ejecuta migraciones al
arrancar; no probar una compilacion contra produccion. El volcado declara
db_version 19 y la migracion 19 finaliza la secuencia.

El directorio de trabajo debe contener config.lua y data/, tambien en servicios
Windows y systemd. Mantener items.otb, el mapa y sus archivos de casas/spawns.
Los puertos configurados son 7171 y 7172; la IP incluida es local.

## Verificacion

1. Configuracion y compilacion limpias en ambos sistemas.
2. Resolver todas las DLL o bibliotecas compartidas del nuevo ejecutable.
3. Arranque con base de pruebas y carga del mapa, NPC y scripts.
4. Login, captura, invocacion, ataques, evolucion, boost, fly/surf/ride/dive y tiendas.
5. Guardado y reinicio: comprobar vida, nivel, experiencia, boost y movimientos
   de las pokeballs.

WARNINGS_AS_ERRORS=ON activa comprobacion estricta. ENABLE_PCH=OFF permite
diagnosticar cabeceras sin precompilacion. .github/workflows/build.yml permite
compilar ambos sistemas en GitHub; no sustituye las pruebas dentro del juego.

Pendiente de recuperar: data/npc/Wentworth.xml referencia Wentworth.lua,
que no esta incluido. No se invento un reemplazo.

## Resultado de las comprobaciones locales

El 6 de octubre de 2026 se configuro y compilo Release x64 con MSVC
19.44.35229 y CMake 3.31.6. El resultado esta en build/windows/Release/.
Se comprobaron los imports del ejecutable: lua51.dll, gmp-10.dll,
libmariadb.dll y pugixml.dll; zlib1.dll acompana al conector. Tambien necesita
el runtime de Visual C++ x64 y las bibliotecas del sistema Windows.

Una prueba de arranque con configuracion aislada cargo config.lua y llego
al intento de conexion a una base inexistente. Esto verifica la carga del
ejecutable y sus DLL; no verifica la conexion real, el mapa ni el juego.
No se utilizo la base de datos del proyecto ni se ejecutaron migraciones.

Los 1475 archivos Lua pasan la compilacion de sintaxis con LuaJIT.
La comprobacion de XML no encontro documentos invalidos. Se corrigieron
el parentesis de combustion.lua, el comentario de mark.xml y el nombre
de Simon The Guard en los spawns. Los tiempos de conexion conservan sus
30 segundos mediante una conversion explicita compatible con Boost 1.85.

Linux y los trabajos de GitHub Actions estan pendientes de ejecutar.
Esta maquina no tiene un entorno Linux disponible. El siguiente paso es
compilar en Ubuntu 24.04 y despues probar ambos servidores con una copia
de la base de datos y un cliente compatible, antes de desplegarlos.
