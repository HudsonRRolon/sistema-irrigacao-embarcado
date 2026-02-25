#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// =================== Pinos ===================
const int sensorUmidade = A0;
const int relePin = 8;
const int ledR = 9;
const int ledG = 10;
const int ledB = 11;
const int botaoModoPin = 2;
const int botaoSelecionarPin = 3;

// =================== Menu e Estados ====================
long ultimoTempoPressionadoModo = 0;
long ultimoTempoPressionadoSelecionar = 0;
const long debounceDelay = 200;

enum MenuState {
  INITIAL_WELCOME,
  MAIN_MENU,
  MANUAL_MODE_MENU,
  CONFIRM_MODE,
  CONFIRM_MANUAL_INTENSITY,
  IRRIGATING_MANUAL
};
MenuState estadoMenu = INITIAL_WELCOME;

enum PlantMode {
  SUCULENTAS = 0,
  ERVAS,
  FLORES,
  HORTALICAS,
  MODO_MANUAL
};

// Frases ajustadas para caber no LCD (max 14 caracteres para o nome do modo)
String modosRegas[] = { "Cactos", "Ervas", "Flores", "Hortalicas", "Modo Manual" }; // Hortali. para Hortalicas
const int numModos = sizeof(modosRegas) / sizeof(modosRegas[0]);

// Frases ajustadas para intensidade manual (max 8 caracteres)
String intensidadesManuais[] = { "Pouca", "Media", "Muita" };
const int numIntensidadesManuais = sizeof(intensidadesManuais) / sizeof(intensidadesManuais[0]);

int modoAtualIndex = SUCULENTAS;
int intensidadeManualIndex = 0;
int modoDeRegaAtivo = -1; // -1 significa nenhum modo automático ativo

// ================= Caracteres personalizados =================
byte seta[8] = {
  0b00000, 0b00100, 0b01110, 0b10101, 0b00100, 0b00100, 0b00000, 0b00000
};

byte gotaAnimacao[4][8] = {
  { 0b00000, 0b00000, 0b00100, 0b01110, 0b11111, 0b11111, 0b01110, 0b00000 },
  { 0b00000, 0b00100, 0b01110, 0b11111, 0b11111, 0b01110, 0b00000, 0b00000 },
  { 0b00000, 0b00000, 0b00000, 0b00100, 0b01110, 0b11111, 0b01110, 0b00000 },
  { 0b00000, 0b00000, 0b00000, 0b00000, 0b00100, 0b01010, 0b00100, 0b00000 }
};

// ================= Tempo =================
unsigned long tempoInicioIrrigacaoManual = 0;
unsigned long duracaoIrrigacaoManual = 0;
unsigned long ultimoTempoLeituraUmidade = 0;
const unsigned long intervaloLeituraUmidade = 5000; // 5 segundos
unsigned long tempoExibicaoMensagem = 0;
const unsigned long duracaoMensagem = 1500;
const unsigned long duracaoWelcome = 2000;

unsigned long ultimoTempoAnimacaoGota = 0;
int frameGota = 0;
const unsigned long intervaloAnimacaoGota = 250;

// ================= Sensor Umidade =================
const int SENSOR_UMIDO_ADC = 500;
const int SENSOR_SECO_ADC = 1000;

const int LIMIARES_UMIDADE[] = { 20, 40, 50, 60 };

// ================= SETUP =================
void setup() {
  Serial.begin(9600);
  lcd.init();
  lcd.backlight();

  lcd.createChar(0, seta);
  lcd.createChar(1, gotaAnimacao[0]);
  lcd.createChar(2, gotaAnimacao[1]);
  lcd.createChar(3, gotaAnimacao[2]);
  lcd.createChar(4, gotaAnimacao[3]);

  pinMode(sensorUmidade, INPUT);
  pinMode(relePin, OUTPUT);
  pinMode(ledR, OUTPUT);
  pinMode(ledG, OUTPUT);
  pinMode(ledB, OUTPUT);

  pinMode(botaoModoPin, INPUT_PULLUP);
  pinMode(botaoSelecionarPin, INPUT_PULLUP);

  desligarIrrigacao();

  lcd.setCursor(0, 0);
  lcd.print("Bem-vindo!");
  lcd.setCursor(0, 1);
  lcd.print("Horta Autonoma");
  tempoExibicaoMensagem = millis();
  estadoMenu = INITIAL_WELCOME;
}

// ================= LOOP =================
void loop() {
  if (estadoMenu == INITIAL_WELCOME) {
    if (millis() - tempoExibicaoMensagem >= duracaoWelcome) {
      estadoMenu = MAIN_MENU;
      exibirMenuPrincipal();
    }
  } else {
    lerBotoes();
    controlarEstadosMenu();
    controlarIrrigacaoAutomatica();
    controlarIrrigacaoManualPorTempo();
  }
}

// ============ FUNÇÕES DE INTERAÇÃO ================
void lerBotoes() {
  int leituraModo = digitalRead(botaoModoPin);
  if (leituraModo == LOW && millis() - ultimoTempoPressionadoModo > debounceDelay) {
    ultimoTempoPressionadoModo = millis();

    if (estadoMenu == MAIN_MENU) {
      modoAtualIndex = (modoAtualIndex + 1) % numModos;
      exibirMenuPrincipal();
      modoDeRegaAtivo = -1;
      desligarIrrigacao();
    } else if (estadoMenu == MANUAL_MODE_MENU) {
      intensidadeManualIndex = (intensidadeManualIndex + 1) % numIntensidadesManuais;
      exibirIntensidadeManual();
    }
  }

  int leituraSelecionar = digitalRead(botaoSelecionarPin);
  if (leituraSelecionar == LOW && millis() - ultimoTempoPressionadoSelecionar > debounceDelay) {
    ultimoTempoPressionadoSelecionar = millis();

    if (estadoMenu == MAIN_MENU) {
      if (modoAtualIndex == MODO_MANUAL) {
        estadoMenu = MANUAL_MODE_MENU;
        intensidadeManualIndex = 0;
        exibirIntensidadeManual();
        modoDeRegaAtivo = -1;
        desligarIrrigacao();
      } else {
        estadoMenu = CONFIRM_MODE;
        tempoExibicaoMensagem = millis();
        exibirConfirmacaoModo();
        modoDeRegaAtivo = modoAtualIndex;
      }
    } else if (estadoMenu == MANUAL_MODE_MENU) {
      estadoMenu = CONFIRM_MANUAL_INTENSITY;
      tempoExibicaoMensagem = millis();
      exibirConfirmacaoIntensidadeManual();
      modoDeRegaAtivo = -1;
    }
  }
}

// =================== OUTRAS FUNÇÕES ===================
void controlarEstadosMenu() {
  if (estadoMenu == CONFIRM_MODE || estadoMenu == CONFIRM_MANUAL_INTENSITY) {
    if (millis() - tempoExibicaoMensagem >= duracaoMensagem) {
      estadoMenu = MAIN_MENU;
      exibirMenuPrincipal();
    }
  }
}

void controlarIrrigacaoAutomatica() {
  if (modoDeRegaAtivo != -1 && modoDeRegaAtivo != MODO_MANUAL && !irrigacaoManualEmAndamento()) {
    if (millis() - ultimoTempoLeituraUmidade >= intervaloLeituraUmidade) {
      int umidade = map(analogRead(sensorUmidade), SENSOR_SECO_ADC, SENSOR_UMIDO_ADC, 0, 100);
      Serial.print("Umidade: ");
      Serial.print(umidade);
      Serial.println("%");

      if (modoDeRegaAtivo >= 0 && modoDeRegaAtivo < (sizeof(LIMIARES_UMIDADE) / sizeof(LIMIARES_UMIDADE[0]))) {
        if (umidade < LIMIARES_UMIDADE[modoDeRegaAtivo]) {
          ligarIrrigacao();
          lcd.setCursor(0, 1);
          lcd.print("Regando!        "); // Limpa a linha
          lcd.setCursor(0,1);
          lcd.print("Regando ");

          if (millis() - ultimoTempoAnimacaoGota >= intervaloAnimacaoGota) {
            ultimoTempoAnimacaoGota = millis();
            frameGota = (frameGota + 1) % 4;
            lcd.setCursor(8, 1);
            lcd.write(byte(frameGota + 1));
          }
          lcd.setCursor(10, 1);
          lcd.print(umidade);
          lcd.print("% ");

        } else {
          desligarIrrigacao();
          lcd.setCursor(0, 1);
          lcd.print("Umidade OK!     ");
          lcd.setCursor(12, 1);
          lcd.print(umidade);
          lcd.print("%");
          ultimoTempoAnimacaoGota = 0;
          frameGota = 0;
        }
      }
      ultimoTempoLeituraUmidade = millis();
    }
  } else if (estadoMenu == MAIN_MENU && !irrigacaoManualEmAndamento()) {
    // Quando no MAIN_MENU e sem irrigação manual ou automática em andamento,
    // a linha 1 mostra a opção selecionada com a seta.
    // A linha 0 é fixa em "Selecione Modo" (ver exibirMenuPrincipal)
  }
}

void controlarIrrigacaoManualPorTempo() {
  if (estadoMenu == IRRIGATING_MANUAL) {
    unsigned long tempoDecorrido = millis() - tempoInicioIrrigacaoManual;
    if (tempoDecorrido >= duracaoIrrigacaoManual) {
      desligarIrrigacao();
      estadoMenu = MAIN_MENU;
      exibirMenuPrincipal();
      modoDeRegaAtivo = -1;
    } else {
      long tempoRestante = (duracaoIrrigacaoManual - tempoDecorrido) / 1000;
      lcd.setCursor(0, 1);
      lcd.print("Faltam: ");
      if (tempoRestante < 10) lcd.print(" ");
      lcd.print(tempoRestante);
      lcd.print("s       ");
    }
  }
}

bool irrigacaoManualEmAndamento() {
  return estadoMenu == IRRIGATING_MANUAL;
}

void ligarIrrigacao() {
  digitalWrite(relePin, HIGH);
  setLedCor(0, 255, 0);
}

void desligarIrrigacao() {
  digitalWrite(relePin, LOW);
  setLedCor(255, 0, 0);
}

void iniciarIrrigacaoManual(unsigned long tempo) {
  ligarIrrigacao();
  tempoInicioIrrigacaoManual = millis();
  duracaoIrrigacaoManual = tempo;
  estadoMenu = IRRIGATING_MANUAL;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Regando Manual!");
}

void setLedCor(int r, int g, int b) {
  analogWrite(ledR, r);
  analogWrite(ledG, g);
  analogWrite(ledB, b);
}

// ===============================================
// FUNÇÕES DE EXIBIÇÃO - AJUSTADAS PARA O NOVO LAYOUT
// ===============================================

void exibirMenuPrincipal() {
  lcd.clear();
  lcd.setCursor(0, 0);
  // Linha 0: Título/Instrução
  if (modoDeRegaAtivo != -1 && modoDeRegaAtivo != MODO_MANUAL) {
    lcd.print("MODO: "); // Indica que um modo está ativo
    lcd.setCursor(6,0);
    // Exibe um pedaço do nome do modo ativo, garantindo que não exceda o limite.
    // O comprimento máximo do nome do modo é 14, então 16 - 12 (pos da substring) = 4 caracteres restantes.
    lcd.print(modosRegas[modoDeRegaAtivo].substring(0, min((int)modosRegas[modoDeRegaAtivo].length(), 10)));
  } else {
    lcd.print("Selecione o modo  "); // Instrução principal
  }


  // Linha 1: Opção atual com seta
  lcd.setCursor(0, 1);
  lcd.write(byte(0)); // Caractere de seta
  lcd.print(" ");
  lcd.print(modosRegas[modoAtualIndex]); // Opção atual
  // Preenche o restante da linha com espaços para limpar
  for (int i = 2 + modosRegas[modoAtualIndex].length(); i < 16; i++) {
    lcd.print(" ");
  }
}

void exibirIntensidadeManual() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Selec. Intens.  "); // Linha superior com instrução

  lcd.setCursor(0, 1);
  lcd.write(byte(0)); // Seta de seleção
  lcd.print(" ");
  lcd.print(intensidadesManuais[intensidadeManualIndex]); // Intensidade selecionada
  lcd.print(" ");
  
  // =========== AQUI ESTÁ A MUDANÇA PARA RECOLOCAR AS GOTAS ===========
  // Posiciona o cursor após o texto da intensidade para as gotas
  int cursor_pos_gotas = 3 + intensidadesManuais[intensidadeManualIndex].length();
  lcd.setCursor(cursor_pos_gotas, 1);
  
  for (int i = 0; i <= intensidadeManualIndex; i++) {
    if (cursor_pos_gotas + i < 16) { // Garante que não escreva fora da tela
      lcd.write(byte(1 + i)); // Usa gotaAnimacao[0], gotaAnimacao[1], gotaAnimacao[2]
    }
  }
  
  // Preenche o restante da linha com espaços para limpar, a partir do último ponto das gotas
  int last_char_pos = cursor_pos_gotas + (intensidadeManualIndex + 1);
  for (int i = last_char_pos; i < 16; i++) {
    lcd.print(" ");
  }
}

void exibirConfirmacaoModo() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Modo: ");
  lcd.print(modosRegas[modoAtualIndex]);
  for (int i = 7 + modosRegas[modoAtualIndex].length(); i < 16; i++) {
    lcd.print(" ");
  }
  lcd.setCursor(0, 1);
  lcd.print("CONFIRMADO!     ");
  Serial.print("Modo selecionado: ");
  Serial.println(modosRegas[modoAtualIndex]);
}

void exibirConfirmacaoIntensidadeManual() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("MAN: ");
  lcd.print(intensidadesManuais[intensidadeManualIndex]);
  for (int i = 5 + intensidadesManuais[intensidadeManualIndex].length(); i < 16; i++) {
    lcd.print(" ");
  }
  lcd.setCursor(0, 1);
  lcd.print("INICIANDO REGA! ");
  Serial.print("Intensidade Manual: ");
  Serial.println(intensidadesManuais[intensidadeManualIndex]);

  if (intensidadeManualIndex == 0) iniciarIrrigacaoManual(3000);
  else if (intensidadeManualIndex == 1) iniciarIrrigacaoManual(5000);
  else iniciarIrrigacaoManual(7000);
}
