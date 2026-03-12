# Snake Game - ESP32

Laboratorio 2 - Sistemas Embebidos

- Jacobo Velásquez Durango

## Descripción

Implementación del videojuego Snake en un microcontrolador ESP32
usando una matriz LED 8x8 bicolor.

## Hardware

- ESP32
- Matriz LED 8x8 bicolor
- 4 pulsadores

## Controles

UP - mover arriba  
DOWN - mover abajo  
LEFT - mover izquierda  
RIGHT - mover derecha  

## Funciones

- Movimiento de serpiente
- Generación de comida
- Detección de colisiones
- Animación Game Over

## Compilación

Proyecto desarrollado usando ESP-IDF.

#include <stdio.h>
#include <stdlib.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "driver/gpio.h"
#include "esp_rom_sys.h"

#define MATRIX_SIZE 8
#define MAX_SNAKE_LENGTH 64

// COLORES
#define OFF 0
#define RED 1
#define GREEN 2

// BOTONES
#define BTN_UP GPIO_NUM_32
#define BTN_DOWN GPIO_NUM_33
#define BTN_LEFT GPIO_NUM_25
#define BTN_RIGHT GPIO_NUM_26

// PINES MATRIZ (EJEMPLO - AJUSTAR DESPUÉS)
int rows[8] = {2,4,5,18,19,21,22,23};

int col_red[8] = {12,13,14,15,16,17,25,26};
int col_green[8] = {27,32,33,34,35,36,39,4};

typedef enum
{
    UP,
    DOWN,
    LEFT,
    RIGHT
} direction_t;

typedef struct
{
    int x;
    int y;
} point_t;

// framebuffer de pantalla
uint8_t matrix[MATRIX_SIZE][MATRIX_SIZE];

// snake
point_t snake[MAX_SNAKE_LENGTH];
int snake_length = 3;

// comida
point_t food;

direction_t dir = RIGHT;

//------------------------------------------------

// CONFIGURACIÓN GPIO

//------------------------------------------------

void gpio_init_all()
{
    gpio_config_t io_conf;

    io_conf.mode = GPIO_MODE_OUTPUT;
    io_conf.pull_down_en = 0;
    io_conf.pull_up_en = 0;

    uint64_t mask = 0;

    for(int i=0;i<8;i++)
        mask |= (1ULL<<rows[i]);

    for(int i=0;i<8;i++)
        mask |= (1ULL<<col_red[i]);

    for(int i=0;i<8;i++)
        mask |= (1ULL<<col_green[i]);

    io_conf.pin_bit_mask = mask;

    gpio_config(&io_conf);

    gpio_config_t btn_conf;

    btn_conf.mode = GPIO_MODE_INPUT;
    btn_conf.pull_up_en = 1;
    btn_conf.pull_down_en = 0;

    btn_conf.pin_bit_mask =
            (1ULL<<BTN_UP) |
            (1ULL<<BTN_DOWN) |
            (1ULL<<BTN_LEFT) |
            (1ULL<<BTN_RIGHT);

    gpio_config(&btn_conf);
}

//------------------------------------------------
// LIMPIAR MATRIZ
//------------------------------------------------

void matrix_clear()
{
    for(int y=0;y<MATRIX_SIZE;y++)
    {
        for(int x=0;x<MATRIX_SIZE;x++)
        {
            matrix[y][x] = OFF;
        }
    }
}

//------------------------------------------------
// DRIVER MATRIZ
//------------------------------------------------

void matrix_draw()
{
    for(int row=0; row<8; row++)
    {
        // apagar filas
        for(int r=0;r<8;r++)
            gpio_set_level(rows[r],0);

        for(int col=0; col<8; col++)
        {
            if(matrix[row][col] == RED)
            {
                gpio_set_level(col_red[col],1);
                gpio_set_level(col_green[col],0);
            }
            else if(matrix[row][col] == GREEN)
            {
                gpio_set_level(col_red[col],0);
                gpio_set_level(col_green[col],1);
            }
            else
            {
                gpio_set_level(col_red[col],0);
                gpio_set_level(col_green[col],0);
            }
        }

        gpio_set_level(rows[row],1);

        esp_rom_delay_us(800);
    }
}

//------------------------------------------------
// GENERAR COMIDA
//------------------------------------------------

void spawn_food()
{
    food.x = rand() % MATRIX_SIZE;
    food.y = rand() % MATRIX_SIZE;
}

//------------------------------------------------
// INICIALIZAR SNAKE
//------------------------------------------------

void snake_init()
{
    snake[0].x = 4;
    snake[0].y = 4;

    snake[1].x = 3;
    snake[1].y = 4;

    snake[2].x = 2;
    snake[2].y = 4;

    spawn_food();
}

//------------------------------------------------
// LEER BOTONES
//------------------------------------------------

void read_buttons()
{
    if(!gpio_get_level(BTN_UP))
        dir = UP;

    if(!gpio_get_level(BTN_DOWN))
        dir = DOWN;

    if(!gpio_get_level(BTN_LEFT))
        dir = LEFT;

    if(!gpio_get_level(BTN_RIGHT))
        dir = RIGHT;
}

//------------------------------------------------
// COLISIONES
//------------------------------------------------

int check_collision(int x, int y)
{
    if(x < 0 || x >= MATRIX_SIZE) return 1;
    if(y < 0 || y >= MATRIX_SIZE) return 1;

    for(int i=0;i<snake_length;i++)
    {
        if(snake[i].x == x && snake[i].y == y)
            return 1;
    }

    return 0;
}

//------------------------------------------------
// MOVER SNAKE
//------------------------------------------------

void snake_move()
{
    point_t new_head = snake[0];

    if(dir == UP) new_head.y--;
    if(dir == DOWN) new_head.y++;
    if(dir == LEFT) new_head.x--;
    if(dir == RIGHT) new_head.x++;

    if(check_collision(new_head.x,new_head.y))
    {
        printf("GAME OVER\n");

        while(1)
        {
            matrix_clear();

            for(int y=0;y<8;y++)
                for(int x=0;x<8;x++)
                    matrix[y][x] = RED;

            matrix_draw();

            vTaskDelay(pdMS_TO_TICKS(300));

            matrix_clear();

            matrix_draw();

            vTaskDelay(pdMS_TO_TICKS(300));
        }
    }

    for(int i=snake_length;i>0;i--)
        snake[i] = snake[i-1];

    snake[0] = new_head;

    if(new_head.x == food.x && new_head.y == food.y)
    {
        snake_length++;

        if(snake_length > MAX_SNAKE_LENGTH)
            snake_length = MAX_SNAKE_LENGTH;

        spawn_food();
    }
    else
    {
        // mover cola
    }
}

//------------------------------------------------
// DIBUJAR JUEGO
//------------------------------------------------

void draw_game()
{
    matrix_clear();

    matrix[food.y][food.x] = RED;

    for(int i=0;i<snake_length;i++)
    {
        matrix[snake[i].y][snake[i].x] = GREEN;
    }
}

//------------------------------------------------
// MAIN
//------------------------------------------------

void app_main()
{
    gpio_init_all();

    snake_init();

    while(1)
    {
        read_buttons();

        snake_move();

        draw_game();

        matrix_draw();

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}
