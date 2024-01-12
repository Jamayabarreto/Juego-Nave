# Juego-Nave
Juego Nave es un juego simple en Python inspirado en el clásico Asteroids, desarrollado utilizando la biblioteca Pygame.

## Instalación

1. Asegúrate de tener Python instalado en tu sistema.
3. https://github.com/Jamayabarreto/Juego-Nave/edit/main/README.md
4. pip install pygame
5. python JuegoNave.py


import pygame, os, platform, random, sys
from pygame.locals import *
from math import *
from pygame import Vector2

if platform.system() == 'Windows':
    os.environ['SDL_VIDEODRIVER'] = 'windib'

# Colores: valores R G B, cuánto de rojo, verde y azul
NEGRO = (0, 0, 0)
BLANCO = (255, 255, 255)
VERDE = (0, 255, 0)
ROSA = (255, 166, 201)

# Inicializa el juego
TAM_PANTALLA = ANCHO_PANTALLA, ALTO_PANTALLA = 640, 480
ANCHO_NAVE, ALTO_NAVE = 17, 21
ANCHO_ASTEROIDE, ALTO_ASTEROIDE = 51, 41
NUM_ASTEROIDES = 3
NUM_ASTEROIDES_DIVIDIR = 5
TASA_AGREGAR_ASTEROIDE = 300
INCREMENTO_VELOCIDAD = 6
DECREMENTO_VELOCIDAD = 13
VELOCIDAD_MAXIMA = 200.0

# Puntos que crean la forma de nuestro asteroide
FORMA_ASTEROIDE = [(5, 0), (0, 15), (0, 30),
                   (15, 40), (20, 30), (30, 40),
                   (45, 30), (45, 25), (25, 20),
                   (45, 10), (36, 0), (25, 5)]

def salir_juego():
    pygame.quit()
    sys.exit()

def presionar_cualquier_tecla():
    while True:
        for evento in pygame.event.get():
            if evento.type == QUIT:
                salir_juego()
            if evento.type == KEYDOWN:
                if evento.key == K_ESCAPE:
                    salir_juego()
                return

def dibujar_texto(texto, fuente, superficie, pos, color=VERDE):
    texto_superficie = fuente.render(texto, 1, color)
    superficie.blit(texto_superficie, pos)

def obtener_explosion():
    explosion = {'velocidad': VELOCIDAD_MAXIMA * 3,
                 'pos': Vector2(200, 150),
                 'rot': 0.0,
                 'surf': pygame.Surface((3, 11))}
    pygame.draw.aaline(explosion['surf'], BLANCO, [1, 0], [1, 10])
    return explosion

def nuevo_asteroide():
    # Estructura de datos para nuestro asteroide
    asteroide = {'velocidad': random.randint(20, 80),
                 'pos': Vector2(random.randint(0, ANCHO_PANTALLA - ANCHO_ASTEROIDE),
                                random.randint(0, ALTO_PANTALLA - ALTO_ASTEROIDE)),
                 'rot': 0.0,
                 'rot_velocidad': random.randint(90, 180) / 1.0,
                 'rot_direccion': random.choice([-1, 1]),
                 'surf': pygame.Surface((ANCHO_ASTEROIDE, ALTO_ASTEROIDE)),
                 'rect': pygame.Rect(0, 0, ANCHO_ASTEROIDE, ALTO_ASTEROIDE),
                 'impactos': 0}

    # Dibuja nuestro asteroide usando formas geométricas
    pygame.draw.polygon(asteroide['surf'], BLANCO, FORMA_ASTEROIDE, 1)
    return asteroide

def nueva_nave():
    # Estructura de datos para nuestra nave espacial, vectorizada
    nave = {'velocidad': 0,
            'pos': Vector2(200, 150),
            'rot': 0.0,
            'rot_velocidad': 360.0,
            'surf': pygame.Surface((ANCHO_NAVE, ALTO_NAVE)),
            'nueva': True}

    # Dibuja la nave espacial usando formas geométricas
    pygame.draw.aaline(nave['surf'], VERDE, [0, 20], [8, 0])
    pygame.draw.aaline(nave['surf'], VERDE, [8, 0], [16, 20])
    pygame.draw.aaline(nave['surf'], VERDE, [2, 15], [7, 15])
    pygame.draw.aaline(nave['surf'], VERDE, [14, 15], [9, 15])
    return nave

def centro_rotar(imagen, ancho, alto):
    """Devuelve la posición de dibujo y hacia dónde se dirige"""
    direccion_x = sin(imagen['rot'] * pi / 180.0) # Convierte grados a radianes y calcula el componente x
    direccion_y = cos(imagen['rot'] * pi / 180.0) # Convierte grados a radianes y calcula el componente y
    return Vector2(imagen['pos'].x - ancho / 2, imagen['pos'].y - alto / 2), Vector2(direccion_x, direccion_y)

## Bucle principal. Inicializa PyGame y nuestro juego ##
def principal():
    pygame.init()
    pantalla = pygame.display.set_mode(TAM_PANTALLA)
    pygame.display.set_caption("Juego Nave")

    fuente_titulo = pygame.font.Font('SFPixelate.ttf', 52)
    fuente_texto = pygame.font.Font('SFPixelate.ttf', 26)
    fuente_puntuacion = pygame.font.Font(None, 72)

    sonido_bienvenida = pygame.mixer.Sound('sfx-01.wav')
    sonido_blaster = pygame.mixer.Sound('blast.wav')
    sonido_asteroide_golpe = pygame.mixer.Sound('hit.wav')
    sonido_colision_jugador = pygame.mixer.Sound('explode.wav')

    rect_pantalla = pantalla.get_rect()
    nave = nueva_nave()
    explosiones = []
    puntuacion = 0
    num_vidas = 3
    tiempo_total_transcurrido_seg = 0
    contador_agregar_asteroide = 0
    en_ejecucion = True

    # Muestra la pantalla de título del juego y espera al usuario
    rect_titulo = fuente_titulo.render('Juego Nave', 1, VERDE).get_rect()
    rect_inicio = fuente_texto.render('Presiona cualquier tecla para comenzar', 1, VERDE).get_rect()
    dibujar_texto('Juego Nave', fuente_titulo, pantalla,
                  (rect_pantalla.centerx - rect_titulo.width / 2,
                   rect_pantalla.centery - (rect_titulo.height + rect_inicio.height + 10) / 2))
    dibujar_texto('Presiona cualquier tecla para comenzar', fuente_texto, pantalla,
                  (rect_pantalla.centerx - rect_inicio.width / 2,
                   rect_pantalla.centery + 10))
    global top_puntuacion
    if top_puntuacion > 0:
        dibujar_texto('Puntuación más alta: %s' % str(top_puntuacion), fuente_texto, pantalla, (20, 10), ROSA)
    pygame.display.update()
    sonido_bienvenida.play()
    presionar_cualquier_tecla()
    reloj = pygame.time.Clock()

    asteroides = []
    for i in range(NUM_ASTEROIDES):
        asteroide = nuevo_asteroide()
        asteroides.append(asteroide)

    ## Bucle de juego ##
    while en_ejecucion:
        for evento in pygame.event.get():
            if evento.type == QUIT:
                salir_juego()

        teclas_presionadas = pygame.key.get_pressed()
        direccion_rotacion = 0.0
        direccion_movimiento = -1

        if teclas_presionadas[K_ESCAPE]:
            salir_juego()
        if teclas_presionadas[K_LEFT]:
            direccion_rotacion += 1.0
        elif teclas_presionadas[K_RIGHT]:
            direccion_rotacion = -1.0
        if teclas_presionadas[K_UP]:
            nave['velocidad'] += INCREMENTO_VELOCIDAD
            if nave['velocidad'] > VELOCIDAD_MAXIMA: nave['velocidad'] = VELOCIDAD_MAXIMA
        elif teclas_presionadas[K_DOWN]:
            nave['velocidad'] -= DECREMENTO_VELOCIDAD
            if nave['velocidad'] < 0: nave['velocidad'] = 0
        if teclas_presionadas[K_SPACE]:
            nueva_explosion = obtener_explosion()
            nueva_explosion['pos'] = Vector2(nave['pos'].x, nave['pos'].y)
            nueva_explosion['rot'] = nave['rot']
            explosiones.append(nueva_explosion)
            sonido_blaster.play()

        pantalla.fill(NEGRO)
        tiempo_transcurrido = reloj.tick(30)
        tiempo_transcurrido_seg = tiempo_transcurrido / 1000.0
        tiempo_total_transcurrido_seg += tiempo_transcurrido_seg

        contador_agregar_asteroide += 1
        if contador_agregar_asteroide == TASA_AGREGAR_ASTEROIDE:
            contador_agregar_asteroide = 0
            asteroide = nuevo_asteroide()
            asteroide['pos'] = Vector2(0 - ANCHO_ASTEROIDE, random.randint(0, ALTO_PANTALLA - ALTO_ASTEROIDE))
            asteroides.append(asteroide)

        ## Actualiza las explosiones ##
        for explosion in explosiones[:]:
            # La primera vez la explosión aún no está rotada
            superficie_explosion_rotada = pygame.transform.rotate(explosion['surf'], explosion['rot'])

            # La superficie rotada puede no devolver las mismas dimensiones que la original
            ancho_explosion, alto_explosion = superficie_explosion_rotada.get_size()

            # Ajustamos x e y para que el centro de la explosión esté en la posición original
            posicion_dibujo_explosion, direccion_explosion = centro_rotar(explosion, ancho_explosion, alto_explosion)
            direccion_explosion *= direccion_movimiento

            # Nueva posición basada en el tiempo
            explosion['pos'] += direccion_explosion * tiempo_transcurrido_seg * explosion['velocidad']

            # Elimina las explosiones que han salido de los bordes de la pantalla de la lista.
            if explosion['pos'].y < 0 and explosion in explosiones:
                explosiones.remove(explosion)
            if explosion['pos'].y + alto_explosion > ALTO_PANTALLA and explosion in explosiones:
                explosiones.remove(explosion)
            if explosion['pos'].x < 0 and explosion in explosiones:
                explosiones.remove(explosion)
            if explosion['pos'].x + ancho_explosion > ANCHO_PANTALLA and explosion in explosiones:
                explosiones.remove(explosion)

            # Comprueba si el enemigo fue golpeado por la explosión y divídelo en dos
            # Elimina de la lista si ya ha sido golpeado varias veces
            rect_explosion = pygame.Rect(posicion_dibujo_explosion.x, posicion_dibujo_explosion.y, ancho_explosion, alto_explosion)
            for asteroide in asteroides[:]:
                if rect_explosion.colliderect(asteroide['rect']) and explosion in explosiones:

                    superficie_asteroide_rotada = pygame.transform.rotate(asteroide['surf'], asteroide['rot'])        
                    ancho_asteroide, alto_asteroide = superficie_asteroide_rotada.get_size()

                    asteroide_mitad = nuevo_asteroide()
                    asteroide_mitad['pos'] = Vector2(asteroide['pos'].x, asteroide['pos'].y)                    

                    asteroide['pos'].y -= alto_asteroide + (alto_asteroide / 2)
                    asteroide_mitad['pos'].y += alto_asteroide + (alto_asteroide / 2)                    

                    asteroide['surf'] = pygame.transform.scale(superficie_asteroide_rotada, (ancho_asteroide - (ancho_asteroide / 4), alto_asteroide - (alto_asteroide / 4)))
                    asteroide_mitad['surf'] = asteroide['surf']                    

                    asteroide['impactos'] += 1

                    if asteroide['impactos'] >= NUM_ASTEROIDES_DIVIDIR:
                        asteroides.remove(asteroide)
                    else:
                        asteroide_mitad['impactos'] = asteroide['impactos']
                        asteroides.append(asteroide_mitad)
                    
                    explosiones.remove(explosion)
                    puntuacion += 100
                    sonido_asteroide_golpe.play()

            pantalla.blit(superficie_explosion_rotada, posicion_dibujo_explosion)

## Actualiza los asteroides, lo mismo que las explosiones ##
        for asteroide in asteroides[:]:
            asteroide_rotado_surf = pygame.transform.rotate(asteroide['surf'], asteroide['rot'])        
            ancho_asteroide, alto_asteroide = asteroide_rotado_surf.get_size()
            asteroide['rot'] += asteroide['rot_direccion'] * asteroide['rot_velocidad'] * tiempo_transcurrido_seg
            asteroide_draw_pos, a_heading = centro_rotar(asteroide, ancho_asteroide, alto_asteroide)        
            asteroide['pos'].x +=1.0 * tiempo_transcurrido_seg * asteroide['velocidad']        
            asteroide['rect'] = pygame.Rect(asteroide_draw_pos.x, asteroide_draw_pos.y, ancho_asteroide, alto_asteroide)

            if asteroide['pos'].y < 0:
                asteroides.remove(asteroide)                
            if asteroide['pos'].y + alto_asteroide > ALTO_PANTALLA:
                asteroides.remove(asteroide)
            if asteroide['pos'].x > ANCHO_PANTALLA + ANCHO_ASTEROIDE:
                asteroide['pos'].x = -ANCHO_ASTEROIDE

            pantalla.blit(asteroide_rotado_surf, asteroide_draw_pos)

        ## Actualiza al jugador, lo mismo que las explosiones ##
        if tiempo_total_transcurrido_seg >= 5:
            nave['nueva'] = False
            tiempo_total_transcurrido_seg = 0            
            
        nave_rotada_surf = pygame.transform.rotate(nave['surf'], nave['rot'])        
        sw, sh = nave_rotada_surf.get_size()
        
        # Rotación de la nave espacial basada en el tiempo
        nave['rot'] += direccion_rotacion * nave['rot_velocidad'] * tiempo_transcurrido_seg
        
        nave_draw_pos, s_heading = centro_rotar(nave, sw, sh)
        s_heading *= direccion_movimiento        
        nave['pos'] += s_heading * tiempo_transcurrido_seg * nave['velocidad']

        # Evita que el jugador salga de los bordes de la pantalla
        if nave['pos'].y < sh:
            nave['pos'].y = sh
        if nave['pos'].y + sh > ALTO_PANTALLA:
            nave['pos'].y = ALTO_PANTALLA - sh
        if nave['pos'].x < sw:
            nave['pos'].x = sw
        if nave['pos'].x + sw > ANCHO_PANTALLA:
            nave['pos'].x = ANCHO_PANTALLA - sw

        # Comprueba si el jugador ha chocado con un enemigo; no lo comprueba durante los primeros 5 segundos
        nave_rect = pygame.Rect(nave_draw_pos.x, nave_draw_pos.y, sw, sh)
        for asteroide in asteroides[:]:
            if nave_rect.colliderect(asteroide['rect']) and not nave['nueva']:
                tiempo_total_transcurrido_seg = 0
                num_vidas -= 1
                nave = nueva_nave()
                sonido_colision_jugador.play()

        # Parpadea al jugador para indicar el tiempo permitido
        if nave['nueva']:
            if tiempo_total_transcurrido_seg > 0.5 and tiempo_total_transcurrido_seg < 1:
                nave_rotada_surf.fill(NEGRO)
            if tiempo_total_transcurrido_seg > 1.5 and tiempo_total_transcurrido_seg < 2:
                nave_rotada_surf.fill(NEGRO)
            if tiempo_total_transcurrido_seg > 2.5 and tiempo_total_transcurrido_seg < 3:
                nave_rotada_surf.fill(NEGRO)
            if tiempo_total_transcurrido_seg > 3.5 and tiempo_total_transcurrido_seg < 4:
                nave_rotada_surf.fill(NEGRO)
            if tiempo_total_transcurrido_seg > 4.5 and tiempo_total_transcurrido_seg < 5:
                nave_rotada_surf.fill(NEGRO)

        pantalla.blit(nave_rotada_surf, nave_draw_pos)            

        ## Muestra la puntuación en el lado izquierdo de la pantalla ##
        dibujar_texto(str(puntuacion), fuente_puntuacion, pantalla, (20, 5), ROSA)

        ## Muestra el número de vidas restantes ##
        x = 28
        for i in range(num_vidas):        
            pantalla.blit(nave['surf'], (x, 60))
            x += ANCHO_NAVE + 10

        # Muestra "Game Over" si no quedan vidas
        if num_vidas <= 0:
            en_ejecucion = False
            pantalla_copia = pantalla.copy()
            rect_pantalla = pantalla.get_rect()
            
            if puntuacion > top_puntuacion:
                top_puntuacion = puntuacion
                rect_top_score = fuente_texto.render('Nueva puntuación más alta: %s!' % str(top_puntuacion), 1, ROSA).get_rect()
                dibujar_texto('Nueva puntuación más alta: %s!' % str(top_puntuacion), fuente_texto, pantalla,
                          (rect_pantalla.centerx - rect_top_score.width / 2, 100))                
                pygame.display.update()
                pygame.time.wait(4000)

            rect_game_over = fuente_titulo.render('¡Juego terminado!', 1, ROSA).get_rect()
            dibujar_texto('¡Juego terminado!', fuente_titulo, pantalla_copia,
                      (rect_pantalla.centerx - rect_game_over.width / 2,
                       rect_pantalla.centery - rect_game_over.height / 2))
            pantalla.blit(pantalla_copia, (0, 0))
            pygame.display.update()
            pygame.time.wait(4000)
                
        pygame.display.update()
        
    principal()

top_puntuacion = 0
if __name__ == '__main__':
    principal()

