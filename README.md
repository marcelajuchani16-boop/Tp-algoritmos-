# Tp-algoritmos-

package org.ayed.tft.ejercito;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.io.PrintWriter;

import org.ayed.tda.vector.VectorDinamico;
import org.ayed.tft.PosicionInvalidaException;
import org.ayed.tft.campeon.Campeon;
import org.ayed.tft.objeto.Objeto;
import org.ayed.tft.rng.RNG;
import org.ayed.tft.tablero.Tablero;
import org.ayed.tft.tienda.Catalogo;
import org.ayed.tft.tienda.OfertaVaciaException;
import org.ayed.tft.tienda.Tienda;

/**
 * Ejército del jugador.
 * Coordina la tienda, el banco, el tablero, el inventario de objetos y el oro.
 */
public class Ejercito {
    private Tienda tienda;
    private Banco banco;
    private Tablero tablero;
    private int oro;
    private int nivel;
    private VectorDinamico<Objeto> inventario;

    public Ejercito(Tienda tienda, Banco banco, Tablero tablero) {
        if (tienda == null || banco == null || tablero == null) {
            throw new IllegalArgumentException("Ningún parámetro puede ser nulo.");
        }
        this.tienda = tienda;
        this.banco = banco;
        this.tablero = tablero;
        this.oro = 0;
        this.nivel = 1;
        this.inventario = new VectorDinamico<>();
    }

    private void validarZonaPropia(int fila, int columna) {
        if (fila < 0 || fila >= tablero.obtenerFilas() / 2) {
            throw new ZonaNoPermitidaException("La celda no pertenece a la mitad del jugador.");
        }
    }


    public int obtenerOro() {
        return oro;
    }

    public void agregarOro(int cantidad) {
        if (cantidad < 0) {
            throw new IllegalArgumentException("La cantidad no puede ser negativa.");
        }
        this.oro += cantidad;
    }

    public int obtenerNivel() {
        return nivel;
    }

    public void subirNivel() {
        this.nivel++;
    }

    public void comprar(int oferta) {
        Campeon candidato = tienda.obtenerOferta(oferta);
        if (candidato == null) {
            throw new OfertaVaciaException("La oferta seleccionada está vacía.");
        }
        int costo = candidato.obtenerRareza().obtenerCosto();
        if (this.oro < costo) {
            throw new OroInsuficienteException("Oro insuficiente.");
        }
        if (banco.obtenerCantidad() >= banco.obtenerCapacidad()) {
            throw new BancoLlenoException("No hay lugar libre en el banco.");
        }

        Campeon comprado = tienda.comprar(oferta);
        this.oro -= costo;
        banco.colocar(comprado);

        // Fusión automática de 3 copias del mismo campeón
        fusionarCopias(comprado.obtenerNombre());
    }

    public void venderDelBanco(int indice) {
        Campeon c = banco.quitar(indice);
        if (c == null) {
            throw new PosicionInvalidaException("No hay campeón en esa posición del banco.");
        }
        int oroRecuperado = c.obtenerRareza().obtenerCosto();
        if (c.obtenerEstrellas() == 2) {
            oroRecuperado *= 3;
        } else if (c.obtenerEstrellas() == 3) {
            oroRecuperado *= 9;
        }

        for (int i = 0; i < c.obtenerCantidadObjetos(); i++) {
            inventario.agregar(c.obtenerObjeto(i));
        }

        this.oro += oroRecuperado;
        tienda.devolver(c);
    }

    public void venderDelTablero(int fila, int columna) {
        validarZonaPropia(fila, columna);
        Campeon c = tablero.quitar(fila, columna);
        if (c == null) {
            throw new PosicionInvalidaException("No hay campeón en esa celda.");
        }
        int oroRecuperado = c.obtenerRareza().obtenerCosto();
        if (c.obtenerEstrellas() == 2) {
            oroRecuperado *= 3;
        } else if (c.obtenerEstrellas() == 3) {
            oroRecuperado *= 9;
        }

        for (int i = 0; i < c.obtenerCantidadObjetos(); i++) {
            inventario.agregar(c.obtenerObjeto(i));
        }

        this.oro += oroRecuperado;
        tienda.devolver(c);
    }

    public void refrescarTienda() {
        int costoRefresco = 2;
        if (this.oro < costoRefresco) {
            throw new OroInsuficienteException("Oro insuficiente para refrescar la tienda.");
        }
        tienda.refrescar();
        this.oro -= costoRefresco;
    }

    public void desplegar(int indiceBanco, int fila, int columna) {
        validarZonaPropia(fila, columna);

        if (tablero.obtenerCantidad() >= nivel) {
            throw new LimiteDeUnidadesException("Límite de unidades desplegadas alcanzado.");
        }
        Campeon c = banco.obtener(indiceBanco);
        if (c == null) {
            throw new PosicionInvalidaException("La casilla del banco está vacía.");
        }
        banco.quitar(indiceBanco);
        try {
            tablero.colocar(fila, columna, c);
        } catch (RuntimeException e) {
            banco.colocar(c);
            throw e;
        }
    }

    public void replegar(int fila, int columna) {
        validarZonaPropia(fila, columna);

        Campeon c = tablero.obtener(fila, columna);
        if (c == null) {
            throw new PosicionInvalidaException("La celda está vacía.");
        }
        banco.colocar(c);
        tablero.quitar(fila, columna);
    }

    
    public void agregarObjeto(Objeto objeto) {
        if (objeto == null) {
            throw new IllegalArgumentException("El objeto no puede ser nulo.");
        }
        inventario.agregar(objeto);
    }

    public void equiparDesdeInventario(int indiceObjeto, int fila, int columna) {
        validarZonaPropia(fila, columna);

        if (indiceObjeto < 0 || indiceObjeto >= inventario.tamanio()) {
            throw new PosicionInvalidaException("Índice fuera de rango.");
        }
        Campeon c = tablero.obtener(fila, columna);
        if (c == null) {
            throw new PosicionInvalidaException("No hay campeón en la celda.");
        }
        Objeto obj = inventario.obtener(indiceObjeto);
        c.equipar(obj);
        inventario.eliminar(indiceObjeto);
    }

    public int obtenerCantidadObjetos() {
        return inventario.tamanio();
    }

    public Objeto obtenerObjeto(int indice) {
        if (indice < 0 || indice >= inventario.tamanio()) {
            throw new PosicionInvalidaException("Índice fuera de rango.");
        }
        return inventario.obtener(indice);
    }

  
    public int obtenerCapacidadDelBanco() {
        return banco.obtenerCapacidad();
    }

    public Campeon obtenerDelBanco(int indice) {
        return banco.obtener(indice);
    }

    public int obtenerFilasDelTablero() {
        return tablero.obtenerFilas();
    }

    public int obtenerColumnasDelTablero() {
        return tablero.obtenerColumnas();
    }

    public Campeon obtenerDelTablero(int fila, int columna) {
        return tablero.obtener(fila, columna);
    }

    public int obtenerCopiasEnReserva(String nombre) {
        return tienda.obtenerCopiasDisponibles(nombre);
    }

    
    public void guardar(String ruta) {
        if (ruta == null || ruta.trim().isEmpty()) {
            throw new IllegalArgumentException("La ruta no puede ser nula ni vacía.");
        }
        try (PrintWriter writer = new PrintWriter(new FileWriter(ruta))) {
            writer.println(this.oro);
            writer.println(this.nivel);

            writer.println(banco.obtenerCapacidad());
            for (int i = 0; i < banco.obtenerCapacidad(); i++) {
                Campeon c = banco.obtener(i);
                if (c != null) {
                    writer.println(i + ";" + c.obtenerNombre() + ";" + c.obtenerEstrellas());
                }
            }
            writer.println("FIN_BANCO");

            writer.println(tablero.obtenerFilas() + ";" + tablero.obtenerColumnas());
            for (int f = 0; f < tablero.obtenerFilas(); f++) {
                for (int c = 0; c < tablero.obtenerColumnas(); c++) {
                    Campeon camp = tablero.obtener(f, c);
                    if (camp != null) {
                        writer.println(f + ";" + c + ";" + camp.obtenerNombre() + ";" + camp.obtenerEstrellas());
                    }
                }
            }
            writer.println("FIN_TABLERO");

            writer.println(inventario.tamanio());
            for (int i = 0; i < inventario.tamanio(); i++) {
                writer.println(inventario.obtener(i).obtenerNombre());
            }
            writer.println("FIN_OBJETOS");
        } catch (IOException e) {
            throw new RuntimeException("Error al guardar la partida: " + e.getMessage(), e);
        }
    }

    public static Ejercito cargar(String ruta, Catalogo catalogo, RNG rng) {
        if (ruta == null || ruta.trim().isEmpty() || catalogo == null || rng == null) {
            throw new IllegalArgumentException("Parámetros no válidos.");
        }

        File archivo = new File(ruta);
        if (!archivo.exists() || !archivo.isFile()) {
            throw new ArchivoNoExistenteException("El archivo no existe: " + ruta);
        }

        try (BufferedReader reader = new BufferedReader(new FileReader(archivo))) {
            int oro = Integer.parseInt(reader.readLine().trim());
            int nivel = Integer.parseInt(reader.readLine().trim());

            Tienda tienda = new Tienda(catalogo, rng);

            int capBanco = Integer.parseInt(reader.readLine().trim());
            Banco banco = new Banco(capBanco);
            String linea;
            while ((linea = reader.readLine()) != null && !linea.equals("FIN_BANCO")) {
                String[] partes = linea.split(";");
                int pos = Integer.parseInt(partes[0].trim());
                String nombre = partes[1].trim();
                int estrellasBanco = Integer.parseInt(partes[2].trim());

                Campeon base = catalogo.obtener(nombre);
                Campeon camp = new Campeon(base);
                for (int e = 1; e < estrellasBanco; e++) {
                    camp.subirEstrella();
                }
                banco.colocar(camp);
            }

            String[] dim = reader.readLine().trim().split(";");
            int filas = Integer.parseInt(dim[0].trim());
            int cols = Integer.parseInt(dim[1].trim());

            Tablero tablero = new Tablero(filas, cols);
            while ((linea = reader.readLine()) != null && !linea.equals("FIN_TABLERO")) {
                String[] partes = linea.split(";");
                int f = Integer.parseInt(partes[0].trim());
                int c = Integer.parseInt(partes[1].trim());
                String nombre = partes[2].trim();
                int estrellasTablero = Integer.parseInt(partes[3].trim());

                Campeon base = catalogo.obtener(nombre);
                Campeon camp = new Campeon(base);
                for (int e = 1; e < estrellasTablero; e++) {
                    camp.subirEstrella();
                }
                tablero.colocar(f, c, camp);
            }

            Ejercito ejercito = new Ejercito(tienda, banco, tablero);
            ejercito.oro = oro;
            ejercito.nivel = nivel;
            return ejercito;

        } catch (IOException e) {
            throw new ArchivoNoExistenteException("Error al leer el archivo: " + e.getMessage());
        }
    }

    
    private void fusionarCopias(String nombreCampeon) {
        for (int estrella = 1; estrella <= 2; estrella++) {
            VectorDinamico<Integer> enBanco = new VectorDinamico<>();
            VectorDinamico<int[]> enTablero = new VectorDinamico<>();

            for (int i = 0; i < banco.obtenerCapacidad(); i++) {
                Campeon c = banco.obtener(i);
                if (c != null && c.obtenerNombre().equals(nombreCampeon) && c.obtenerEstrellas() == estrella) {
                    enBanco.agregar(i);
                }
            }

            for (int f = 0; f < tablero.obtenerFilas() / 2; f++) {
                for (int c = 0; c < tablero.obtenerColumnas(); c++) {
                    Campeon camp = tablero.obtener(f, c);
                    if (camp != null && camp.obtenerNombre().equals(nombreCampeon) && camp.obtenerEstrellas() == estrella) {
                        enTablero.agregar(new int[]{f, c});
                    }
                }
            }

            if (enBanco.tamanio() + enTablero.tamanio() >= 3) {
                Campeon fusionado = null;
                for (int k = 0; k < 3; k++) {
                    Campeon quitado;
                    if (k < enBanco.tamanio()) {
                        quitado = banco.quitar(enBanco.obtener(k));
                    } else {
                        int[] pos = enTablero.obtener(k - enBanco.tamanio());
                        quitado = tablero.quitar(pos[0], pos[1]);
                    }
                    if (fusionado == null) {
                        fusionado = quitado;
                    }

                    for (int obj = 0; obj < quitado.obtenerCantidadObjetos(); obj++) {
                        inventario.agregar(quitado.obtenerObjeto(obj));
                    }
                }
                fusionado.subirEstrella();
                banco.colocar(fusionado);
            }
        }
    }
}
