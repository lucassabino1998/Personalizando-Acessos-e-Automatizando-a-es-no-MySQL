O objetivo deste projeto foi a criação de triggers em um banco de dados de e-commerce para controlar eventos importantes relacionados à manutenção de dados sensíveis, como informações de usuários.
As triggers foram implementadas para garantir a segurança, a rastreabilidade e a integridade das informações durante as operações de DELETE e UPDATE.



USE company_constraints;

-- 1. Número de empregados por departamento e localidade
CREATE OR REPLACE VIEW vw_num_empregados_departamento_localidade AS
SELECT 
    d.Dname,
    l.Dlocation,
    COUNT(e.Ssn) AS qtd_empregados
FROM 
    employee e
JOIN 
    departament d ON e.Dno = d.Dnumber
JOIN 
    dept_locations l ON d.Dnumber = l.Dnumber
GROUP BY 
    d.Dname, l.Dlocation;

-- 2. Lista de departamentos e seus gerentes
CREATE OR REPLACE VIEW vw_departamentos_gerentes AS
SELECT 
    d.Dname,
    e.Fname AS Gerente_Fname,
    e.Lname AS Gerente_Lname
FROM 
    departament d
JOIN 
    employee e ON d.Mgr_ssn = e.Ssn;

-- 3. Projetos com maior número de empregados
CREATE OR REPLACE VIEW vw_projetos_mais_empregados AS
SELECT 
    p.Pname,
    COUNT(w.Essn) AS qtd_empregados
FROM 
    project p
JOIN 
    works_on w ON p.Pnumber = w.Pno
GROUP BY 
    p.Pname
ORDER BY 
    qtd_empregados DESC;

-- 4. Lista de projetos, departamentos e gerentes
CREATE OR REPLACE VIEW vw_projetos_departamentos_gerentes AS
SELECT 
    p.Pname,
    d.Dname AS Departamento,
    e.Fname AS Gerente_Fname,
    e.Lname AS Gerente_Lname
FROM 
    project p
JOIN 
    departament d ON p.Dnum = d.Dnumber
JOIN 
    employee e ON d.Mgr_ssn = e.Ssn;

-- 5. Quais empregados possuem dependentes e se são gerentes
CREATE OR REPLACE VIEW vw_empregados_dependentes_gerentes AS
SELECT 
    e.Fname,
    e.Lname,
    CASE WHEN d.Essn IS NOT NULL THEN 'Sim' ELSE 'Não' END AS Possui_Dependentes,
    CASE WHEN e.Ssn IN (SELECT Mgr_ssn FROM departament) THEN 'Sim' ELSE 'Não' END AS Eh_Gerente
FROM 
    employee e
LEFT JOIN 
    dependent d ON e.Ssn = d.Essn
GROUP BY 
    e.Ssn;
-- Criar usuário gerente
CREATE USER 'gerente'@'localhost' IDENTIFIED BY 'senha123';
GRANT SELECT ON company_constraints.employee TO 'gerente'@'localhost';
GRANT SELECT ON company_constraints.departament TO 'gerente'@'localhost';
GRANT SELECT ON company_constraints.vw_departamentos_gerentes TO 'gerente'@'localhost';
GRANT SELECT ON company_constraints.vw_projetos_departamentos_gerentes TO 'gerente'@'localhost';
GRANT SELECT ON company_constraints.vw_empregados_dependentes_gerentes TO 'gerente'@'localhost';

-- Criar usuário empregado
CREATE USER 'empregado'@'localhost' IDENTIFIED BY 'senha123';
GRANT SELECT ON company_constraints.employee TO 'empregado'@'localhost';
GRANT SELECT ON company_constraints.vw_num_empregados_departamento_localidade TO 'empregado'@'localhost';
GRANT SELECT ON company_constraints.vw_projetos_mais_empregados TO 'empregado'@'localhost';

-- Revogar acesso de empregado a tabelas de departamento
REVOKE SELECT ON company_constraints.departament FROM 'empregado'@'localhost';

USE ecommerce_db; -- ou seu banco de e-commerce

-- Criar tabela de backup
CREATE TABLE IF NOT EXISTS deleted_users_backup (
    id INT,
    nome VARCHAR(100),
    email VARCHAR(100),
    data_exclusao DATETIME
);

-- Criar trigger para antes do delete
DELIMITER //

CREATE TRIGGER before_delete_user
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
    INSERT INTO deleted_users_backup (id, nome, email, data_exclusao)
    VALUES (OLD.id, OLD.nome, OLD.email, NOW());
END//

DELIMITER ;
DELIMITER //

CREATE TRIGGER before_update_colaboradores
BEFORE UPDATE ON colaboradores
FOR EACH ROW
BEGIN
    IF OLD.salario_base <> NEW.salario_base THEN
        SET NEW.salario_base = NEW.salario_base * 1.05; -- aplica aumento automático de 5%
    END IF;
END//

DELIMITER ;
