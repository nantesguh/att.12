# att.12
import 'package:flutter/material.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';

// Modelo de dados do Planeta
class Planet {
  final int? id;
  final String name;
  final double distanceFromSun;
  final double size;
  final String? nickname;

  Planet({this.id, required this.name, required this.distanceFromSun, required this.size, this.nickname});

  // Converte um planeta em um mapa para armazenar no banco
  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'name': name,
      'distanceFromSun': distanceFromSun,
      'size': size,
      'nickname': nickname,
    };
  }

  // Converte um mapa para um planeta
  factory Planet.fromMap(Map<String, dynamic> map) {
    return Planet(
      id: map['id'],
      name: map['name'],
      distanceFromSun: map['distanceFromSun'],
      size: map['size'],
      nickname: map['nickname'],
    );
  }
}

// Banco de dados e operações CRUD
class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();
  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('planets.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);
    return await openDatabase(path, version: 1, onCreate: _createDB);
  }

  Future _createDB(Database db, int version) async {
    const idType = 'INTEGER PRIMARY KEY AUTOINCREMENT';
    const textType = 'TEXT NOT NULL';
    const realType = 'REAL NOT NULL';

    await db.execute('''
      CREATE TABLE planets (
        id $idType,
        name $textType,
        distanceFromSun $realType,
        size $realType,
        nickname $textType
      )
    ''');
  }

  Future<int> insertPlanet(Planet planet) async {
    final db = await instance.database;
    return await db.insert('planets', planet.toMap());
  }

  Future<List<Planet>> getPlanets() async {
    final db = await instance.database;
    final result = await db.query('planets');
    return result.map((e) => Planet.fromMap(e)).toList();
  }

  Future<Planet?> getPlanet(int id) async {
    final db = await instance.database;
    final result = await db.query('planets', where: 'id = ?', whereArgs: [id]);
    if (result.isNotEmpty) {
      return Planet.fromMap(result.first);
    }
    return null;
  }

  Future<int> updatePlanet(Planet planet) async {
    final db = await instance.database;
    return await db.update(
      'planets',
      planet.toMap(),
      where: 'id = ?',
      whereArgs: [planet.id],
    );
  }

  Future<int> deletePlanet(int id) async {
    final db = await instance.database;
    return await db.delete(
      'planets',
      where: 'id = ?',
      whereArgs: [id],
    );
  }
}

// Tela inicial com a lista de planetas
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Planetas CRUD',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: PlanetListScreen(),
    );
  }
}

// Tela de lista de planetas
class PlanetListScreen extends StatefulWidget {
  @override
  _PlanetListScreenState createState() => _PlanetListScreenState();
}

class _PlanetListScreenState extends State<PlanetListScreen> {
  late Future<List<Planet>> planets;

  @override
  void initState() {
    super.initState();
    planets = DatabaseHelper.instance.getPlanets();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Planetas'),
      ),
      body: FutureBuilder<List<Planet>>(
        future: planets,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return Center(child: CircularProgressIndicator());
          } else if (snapshot.hasError) {
            return Center(child: Text('Erro ao carregar planetas.'));
          } else if (snapshot.data == null || snapshot.data!.isEmpty) {
            return Center(child: Text('Nenhum planeta cadastrado.'));
          } else {
            final planetList = snapshot.data!;
            return ListView.builder(
              itemCount: planetList.length,
              itemBuilder: (context, index) {
                final planet = planetList[index];
                return ListTile(
                  title: Text(planet.name),
                  subtitle: Text(planet.nickname ?? 'Sem apelido'),
                  onTap: () => Navigator.push(
                    context,
                    MaterialPageRoute(
                      builder: (context) => PlanetDetailScreen(planet: planet),
                    ),
                  ),
                );
              },
            );
          }
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          Navigator.push(
            context,
            MaterialPageRoute(builder: (context) => PlanetFormScreen()),
          );
        },
        child: Icon(Icons.add),
      ),
    );
  }
}

// Tela de detalhes do planeta
class PlanetDetailScreen extends StatelessWidget {
  final Planet planet;

  PlanetDetailScreen({required this.planet});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(planet.name),
      ),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Nome: ${planet.name}'),
            Text('Distância do Sol: ${planet.distanceFromSun} AU'),
            Text('Tamanho: ${planet.size} km'),
            if (planet.nickname != null) Text('Apelido: ${planet.nickname}'),
          ],
        ),
      ),
    );
  }
}

// Tela de formulário para adicionar/editar planeta
class PlanetFormScreen extends StatefulWidget {
  @override
  _PlanetFormScreenState createState() => _PlanetFormScreenState();
}

class _PlanetFormScreenState extends State<PlanetFormScreen> {
  final _formKey = GlobalKey<FormState>();
  final _nameController = TextEditingController();
  final _distanceController = TextEditingController();
  final _sizeController = TextEditingController();
  final _nicknameController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Cadastrar Planeta')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                controller: _nameController,
                decoration: InputDecoration(labelText: 'Nome do Planeta'),
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Este campo é obrigatório';
                  }
                  return null;
                },
              ),
              TextFormField(
                controller: _distanceController,
                decoration: InputDecoration(labelText: 'Distância do Sol (AU)'),
                keyboardType: TextInputType.number,
                validator: (value) {
                  if (value == null || value.isEmpty || double.tryParse(value) == null || double.parse(value) <= 0) {
                    return 'Por favor, insira um valor numérico positivo';
                  }
                  return null;
                },
              ),
              TextFormField(
                controller: _sizeController,
                decoration: InputDecoration(labelText: 'Tamanho (km)'),
                keyboardType: TextInputType.number,
                validator: (value) {
                  if (value == null || value.isEmpty || double.tryParse(value) == null || double.parse(value) <= 0) {
                    return 'Por favor, insira um valor numérico positivo';
                  }
                  return null;
                },
              ),
              TextFormField(
                controller: _nicknameController,
                decoration: InputDecoration(labelText: 'Apelido (opcional)'),
              ),
              SizedBox(height: 20),
              ElevatedButton(
                onPressed: () {
                  if (_formKey.currentState!.validate()) {
                    final planet = Planet(
                      name: _nameController.text,
                      distanceFromSun: double.parse(_distanceController.text),
                      size: double.parse(_sizeController.text),
                      nickname: _nicknameController.text.isEmpty ? null : _nicknameController.text,
                    );

                    DatabaseHelper.instance.insertPlanet(planet);
                    Navigator.pop(context);
                  }
                },
                child: Text('Salvar'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
