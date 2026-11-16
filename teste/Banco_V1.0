-- =====================================================================
-- Rastor — esquema do banco (MySQL 8.0.19+)   versão do esquema: 0.8.0
-- Rodar UMA vez com o usuário rastor_admin.
--
-- Antes: no servidor, default_time_zone = '+00:00'.
-- Em TODA conexão do app: SET time_zone = '+00:00';
--   (os DEFAULT CURRENT_TIMESTAMP abaixo dependem disso para gravar UTC)
--
-- Regras de projeto:
--  * Todo dado pessoal aponta para usuarios e some junto (ON DELETE CASCADE).
--  * Todo horário é UTC. A conversão para o fuso da pessoa fica no Python (zoneinfo).
--  * Apagar é DELETE de verdade (sem coluna "apagado").
--  * Texto sensível (senha, cartão, documento) é mascarado NO PYTHON antes de
--    gravar em conversas e treino_desconhecidas. O banco não checa isso.
-- =====================================================================

CREATE DATABASE IF NOT EXISTS rastor
  CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;   -- MariaDB: utf8mb4_unicode_ci
USE rastor;

-- ---------------------------------------------------------------- usuários
CREATE TABLE usuarios (
  id             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  nome           VARCHAR(80)  NOT NULL,
  email          VARCHAR(255) NOT NULL,
  senha_hash     VARCHAR(255) NULL,                 -- Argon2/bcrypt; NULL se entrar com Google/Apple
  fuso           VARCHAR(40)  NOT NULL DEFAULT 'America/Sao_Paulo',
  memoria_ligada BOOLEAN      NOT NULL DEFAULT 1,
  voz_ligada     BOOLEAN      NOT NULL DEFAULT 0,   -- a voz começa desligada
  criado_em      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_usuarios_email (email)
) ENGINE=InnoDB;

-- LGPD: o que a pessoa aceitou, e quando
CREATE TABLE consentimentos (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id   BIGINT UNSIGNED NOT NULL,
  tipo         ENUM('termos','memoria','voz','treino') NOT NULL,
  aceito       BOOLEAN     NOT NULL,
  versao_texto VARCHAR(20) NOT NULL,                -- versão do documento aceito
  criado_em    DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_consent_usuario (usuario_id, tipo, criado_em),
  CONSTRAINT fk_consent_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ---------------------------------------------------------------- dados do usuário
CREATE TABLE lugares (                               -- futuro Olho de Deus
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id BIGINT UNSIGNED NOT NULL,
  nome       VARCHAR(120) NOT NULL,
  latitude   DECIMAL(9,6) NOT NULL,
  longitude  DECIMAL(9,6) NOT NULL,
  criado_em  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_lugares_usuario (usuario_id),
  CONSTRAINT fk_lugares_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE eventos (                               -- Agenda.md
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id BIGINT UNSIGNED NOT NULL,
  titulo     VARCHAR(255) NOT NULL,
  inicio     DATETIME NOT NULL,                      -- UTC
  fim        DATETIME NULL,
  lugar_id   BIGINT UNSIGNED NULL,                   -- apagar o lugar não apaga o evento
  criado_em  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_eventos_usuario_inicio (usuario_id, inicio),
  CONSTRAINT fk_eventos_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE,
  CONSTRAINT fk_eventos_lugar FOREIGN KEY (lugar_id)
    REFERENCES lugares (id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE tarefas (                               -- Tarefas.md
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id   BIGINT UNSIGNED NOT NULL,
  titulo       VARCHAR(255) NOT NULL,
  feita        BOOLEAN  NOT NULL DEFAULT 0,
  concluida_em DATETIME NULL,
  criado_em    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_tarefas_usuario (usuario_id, feita, criado_em),
  CONSTRAINT fk_tarefas_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE lembretes (                             -- Lembretes.md
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id BIGINT UNSIGNED NOT NULL,
  texto      VARCHAR(255) NOT NULL,
  avisar_em  DATETIME NOT NULL,                      -- UTC
  avisado    BOOLEAN  NOT NULL DEFAULT 0,            -- [x] do cofre = "já avisei"
  avisado_em DATETIME NULL,
  criado_em  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_lembretes_fila (avisado, avisar_em),        -- agendador: vencidos e não avisados
  KEY ix_lembretes_usuario (usuario_id, avisado, avisar_em),
  CONSTRAINT fk_lembretes_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE memoria (                               -- Memoria/*.md
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id    BIGINT UNSIGNED NOT NULL,
  chave         VARCHAR(40)  NULL,                   -- 'nome'; NULL para fato solto
  texto         VARCHAR(255) NOT NULL,               -- só o que o usuário disse
  canal         ENUM('conversa','tela') NOT NULL,    -- por onde o usuário informou
  origem_texto  VARCHAR(255) NOT NULL,               -- a frase original (já sem segredos)
  criado_em     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  atualizado_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_memoria_chave (usuario_id, chave),   -- vários NULL são permitidos
  KEY ix_memoria_usuario (usuario_id, criado_em),
  CONSTRAINT fk_memoria_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ---------------------------------------------------------------- conversa e estado
CREATE TABLE conversas (                             -- Conversas/AAAA-MM-DD.md
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id   BIGINT UNSIGNED NOT NULL,
  fala_usuario TEXT NOT NULL,                        -- sensível já mascarado
  resposta     TEXT NOT NULL,
  intencao     VARCHAR(40) NOT NULL,
  confianca    FLOAT NOT NULL,                       -- baixa = candidata a virar exemplo
  criado_em    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_conversas_usuario (usuario_id, criado_em),
  KEY ix_conversas_duvida (confianca),
  CONSTRAINT fk_conversas_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Substitui o self.pendente do Cerebro: uma pendência por sessão.
-- tipo e dados espelham o dicionário atual:
--   confirmar     {"acao":"apagar_evento","rotulo":"...","id":7}
--   texto_tarefa  {}
--   texto_memoria {}
--   marcar        {"o_que":"agendar","h":15,"m":0,"data":"2026-11-20"}
CREATE TABLE pendencias (
  usuario_id BIGINT UNSIGNED NOT NULL,
  sessao     VARCHAR(40) NOT NULL DEFAULT 'padrao',
  tipo       ENUM('confirmar','texto_tarefa','texto_memoria','marcar') NOT NULL,
  dados      JSON NOT NULL,
  expira_em  DATETIME NOT NULL,                      -- ~5 minutos
  PRIMARY KEY (usuario_id, sessao),                  -- grava com REPLACE / ON DUPLICATE KEY
  KEY ix_pendencias_expira (expira_em),
  CONSTRAINT fk_pendencias_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- ---------------------------------------------------------------- treino e controle
CREATE TABLE treino_exemplos (                       -- Treino/Exemplos.md
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id BIGINT UNSIGNED NULL,                   -- NULL = vale para todos; preenchido = pessoal
  intencao   VARCHAR(40)  NOT NULL,
  frase      VARCHAR(255) NOT NULL,
  origem     ENUM('semente','usuario') NOT NULL DEFAULT 'usuario',
  criado_em  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_treino_intencao (intencao),
  KEY ix_treino_usuario (usuario_id),
  CONSTRAINT fk_treino_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;
-- Carregue com ORDER BY id: a validação cruzada (i % k) depende da ordem.

CREATE TABLE treino_desconhecidas (                  -- Treino/Desconhecidas.md
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  usuario_id BIGINT UNSIGNED NOT NULL,
  frase      VARCHAR(255) NOT NULL,                  -- sensível já mascarado
  resolvida  BOOLEAN  NOT NULL DEFAULT 0,
  criado_em  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY ix_desconhecidas_fila (resolvida, criado_em),
  CONSTRAINT fk_desconhecidas_usuario FOREIGN KEY (usuario_id)
    REFERENCES usuarios (id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE modelo_versoes (                        -- histórico de taxa_de_acerto()
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  qtd_exemplos  INT   NOT NULL,
  acuracia      FLOAT NOT NULL,
  criado_em     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB;

CREATE TABLE esquema_versoes (                       -- migrações aplicadas
  versao      VARCHAR(20) NOT NULL,
  aplicado_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (versao)
) ENGINE=InnoDB;
INSERT INTO esquema_versoes (versao) VALUES ('0.8.0');

-- ---------------------------------------------------------------- usuário do app
-- Troque a senha (e guarde em RASTOR_DB_PASS, nunca no código).
CREATE USER IF NOT EXISTS 'rastor_app'@'localhost' IDENTIFIED BY 'TROQUE_ESTA_SENHA';
GRANT SELECT, INSERT, UPDATE, DELETE ON rastor.* TO 'rastor_app'@'localhost';

-- ---------------------------------------------------------------- rotinas (agendador / limpeza)
-- A cada minuto: lembretes vencidos (só envie a notificação se o UPDATE afetar 1 linha)
--   SELECT id, usuario_id, texto FROM lembretes
--    WHERE avisado = 0 AND avisar_em <= UTC_TIMESTAMP();
--   UPDATE lembretes SET avisado = 1, avisado_em = UTC_TIMESTAMP()
--    WHERE id = %s AND avisado = 0;
-- A cada hora: limpar pendências vencidas
--   DELETE FROM pendencias WHERE expira_em < UTC_TIMESTAMP();
-- Retenção: lembretes avisados há mais de 90 dias
--   DELETE FROM lembretes WHERE avisado = 1 AND avisado_em < UTC_TIMESTAMP() - INTERVAL 90 DAY;
