# msi2lmp
***This tool is used to transform .car to .dat of MD simulation file.



diaoyong.py
This script is used to call the conversion tool of msi2lmp.exe. 
Unzip msi2lmp to the current folder.
Enter the address of the .car file and the .mdf file, and click "生成data文件" 
to get two files. x.data and data.x, where x.data is the converted atomic coordinate file.
 
edited by Marcoliviz
2020/3/10

-------------------------------------------------------------------------------------
CHANGELOG
 
Some users install the win version of lammps, which will cause conflicts in the force 
field path. A new version has been edited to resolve the conflict
 
2020/4/9
 
-------------------------------------------------------------------------------------
CHANGELOG
 
The -n parameter is omitted to avoid coordinate conversion errors.
 
2021/12/9
